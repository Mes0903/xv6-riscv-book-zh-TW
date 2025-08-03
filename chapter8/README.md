---
title: xv6 riscv book chapter 8：File system
date: 2025-08-04
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---

# xv6 riscv book chapter 8：File system

檔案系統的目的是為了組織與儲存資料。 檔案系統通常也會支援使用者與應用程式之間的資料共享，並具備資料的持久性，也就是在重新開機後資料仍然可用。 xv6 的檔案系統提供類 Unix 的檔案、目錄與路徑名稱（詳見第一章），並透過 virtio 硬碟來保存其資料以達成持久性。 這個檔案系統需要面對數個挑戰：

- 檔案系統需要在硬碟上建立資料結構，來表示具名目錄與檔案所組成的樹狀結構，記錄每個檔案內容所使用的區塊位置，並追蹤硬碟中哪些區域尚未被使用
- 檔案系統必須支援當機復原（crash recovery）。 也就是說，如果系統當機（例如電源故障）時中斷了操作，重新啟動後檔案系統仍必須能夠正常運作。 當機的風險在於，它可能會中斷一連串更新操作，使硬碟上的資料結構處於不一致的狀態（例如某個區塊既被某個檔案使用，又同時被標記為未使用）
- 不同的 process 可能會同時操作檔案系統，因此檔案系統的程式碼必須協調彼此之間的動作，以維護資料的一致性與不變性條件
- 存取硬碟的速度遠比存取記憶體慢上好幾個數量級，因此檔案系統必須在記憶體中維護一個常用區塊的快取

本章接下來將說明 xv6 是如何解決這些挑戰的

## 8.1 Overview

xv6 的檔案系統實作被劃分為七個層次，如圖 8.1 所示。 最底層是硬碟層，負責對 virtio 硬碟進行區塊的讀寫。 buffer cache 層會對磁碟區塊進行快取，並協調對其的存取，確保同一時間只有一個 kernel process 可以修改某個特定區塊中的資料。 logging 層允許更高層級的程式將多個區塊的更新包裝成一個 transaction，並保證在系統當機的情況下，這些更新能夠以原子地方式完成（也就是要麼全部更新，要麼全部不更新）

inode 層提供單一檔案的表示方式，每個檔案由一個 inode 表示，具有唯一的 i-number，以及一些儲存檔案內容的區塊。 directory 層把每個目錄實作成一種特殊的 inode，它的內容是一串目錄項目，每個項目包含一個檔案名稱與對應的 i-number。 pathname 層提供階層式路徑名稱，例如 `/usr/rtm/xv6/fs.c`，並透過遞迴查找來解析這些路徑。 file descriptor 層則將許多 Unix 資源（例如 pipe、裝置、檔案等）抽象成使用檔案系統介面的方式，讓應用程式開發者的工作變得更簡單

![（Figure 8.1: Layers of the xv6 file system.）](image/fslayer.png)

硬碟硬體傳統上會將磁碟上的資料呈現為一個個 512-byte block（也稱為 sector）的編號序列：sector 0 是最前面的 512 bytes，sector 1 是接下來的 512 bytes，以此類推。 作業系統在檔案系統中使用的區塊大小可以與硬碟的 sector 大小不同，但通常會是 sector 大小的倍數。 xv6 會將讀取進記憶體的磁碟區塊儲存在型別為 `struct buf` 的物件中。 這些結構中的資料有時可能與實際磁碟上的資料不同步：例如資料可能還沒從磁碟中讀取完成（硬碟還在讀取中但尚未返回該 sector 的內容），或者資料已被軟體修改了但尚未寫回磁碟

檔案系統必須規劃好要將 inode 與資料區塊存放在哪些磁碟位置。 為了達成這點，xv6 將整個磁碟劃分成幾個區段，如圖 8.2 所示。 檔案系統不使用 block 0（因為它是 boot sector）。 block 1 被稱為 superblock，其中包含檔案系統的中繼資料（例如檔案系統的區塊總數、資料區塊的數量、inode 的數量，以及用於 log 的區塊數）

從 block 2 開始是用來存放 log 的區塊。 在 log 區塊之後是 inode 區段，每個區塊會存放多個 inode。 再往後是 bitmap 區塊，用來追蹤哪些資料區塊已被使用。 剩下的區塊就是資料區塊，每一個要麼被 bitmap 標記為 free，要麼用來儲存檔案或目錄的內容。 superblock 會由一個名為 `mkfs` 的獨立程式填寫，它會建立初始的檔案系統

![（Figure 8.2: Structure of the xv6 file system.）](image/fslayout.png)

本章接下來將依序介紹每個層次，從 buffer cache 開始。 請特別注意那些設計良好的底層抽象是如何讓高層的設計變得更簡潔的

## 8.2 Buffer cache layer

buffer cache 有兩個主要任務：第一是同步對磁碟區塊的存取，確保記憶體中對每個區塊只有一個副本，並且同一時間只有一個 kernel 執行緒會使用該副本； 第二是對常用的區塊進行快取，避免每次都要從緩慢的硬碟中重新讀取。 相關程式碼實作位於 `bio.c`

buffer cache 對外提供的主要介面包括 `bread` 與 `bwrite`； 前者會取得一個「buf」，也就是某個磁碟區塊在記憶體中的副本，這份資料可以被讀取或修改，而後者則會將修改過的 buffer 寫回對應的磁碟區塊。 當 kernel 執行緒使用完這個 buffer 後，必須呼叫 `brelse` 來釋放該 buffer。 buffer cache 為每個 buffer 使用一個 sleep-lock，以確保同一時間只有一個執行緒能使用某個 buffer（也就是某個磁碟區塊）； `bread` 會回傳一個已上鎖的 buffer，而 `brelse` 則會釋放該鎖

buffer cache 中只有固定數量的 buffer 可用來儲存磁碟區塊，這表示如果檔案系統要求某個目前不在 cache 中的區塊，buffer cache 就必須回收一個目前用來儲存其他區塊的 buffer。 它會選擇最近最少使用（Least Recently Used, LRU）的那個 buffer 來回收並用於新的區塊。 這基於一個假設：最近最少使用的 buffer 很可能短期內也不會再被使用

## 8.3 Code: Buffer cache

buffer cache 是由多個 buffer 組成的 doubly-linked list。 `main` 函數會呼叫 `binit`（[kernel/main.c:27](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/main.c#L27)），並用靜態陣列 `buf` 中的 `NBUF` 個 buffer 初始化整個 doubly-linked list（[kernel/bio.c:43-52](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L43-L52)）。 在初始化之後，其他所有對 buffer cache 的存取都透過 `bcache.head` 指向的 linked list 來進行，而不再直接使用 `buf` 陣列

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;

void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf + NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```

每個 buffer 都有兩個與其狀態有關的欄位。 `valid` 欄位表示這個 buffer 中目前包含了某個磁碟區塊的副本。 `disk` 欄位則表示該 buffer 的內容已經交由磁碟處理，也就是可能已經被磁碟寫入了（例如從磁碟讀資料到 `data` 陣列中）

```c
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
};
```

`bread`（[kernel/bio.c:93](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L93)）會呼叫 `bget` 來取得對應於某個 sector 的 buffer（[kernel/bio.c:97](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L97)）。 若這個 buffer 需要從磁碟讀取資料，`bread` 會呼叫 `virtio_disk_rw` 來執行讀取，然後才回傳這個 buffer

```c
// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}
```

`bget`（[kernel/bio.c:59](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L59)）會掃描整個 buffer list，尋找一個擁有指定裝置號與 sector 編號的 buffer（[kernel/bio.c:65-73](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L65-L73)）。 若有找到，`bget` 會為該 buffer 取得 sleep-lock，然後回傳這個已上鎖的 buffer

如果該 sector 沒有對應的快取 buffer，`bget` 就必須建一個新的，這可能會重複使用目前儲存其他 sector 的 buffer。 接著它會再掃描一次 buffer list，尋找一個未被使用的 buffer（`b->refcnt = 0`）； 只要符合這個條件就可以使用。 `bget` 接著會更新這個 buffer 的中繼資料，將其裝置編號與 sector 編號改成新的，並且為它上鎖。 特別注意，`b->valid = 0` 這行會強制讓 `bread` 之後從磁碟重新讀取資料，而不是錯誤地使用這個 buffer 之前的內容

每個 sector 最多只能有一個對應的快取 buffer，這點非常重要，因為這樣才能保證讀取端能看到寫入端的更新結果，同時檔案系統也仰賴對 buffer 上鎖來實現同步機制。 `bget` 為了確保這個不變性，會在執行第一次檢查（判斷某個區塊是否已存在於 cache）到第二次回收並設為新的 buffer 的整個過程中，持續持有 `bcache.lock`。 這個過程中會設定該 buffer 的 `dev`、`blockno` 與 `refcnt`。 這種設計可以讓「檢查快取是否存在」與「指定一個 buffer 來存放該區塊」這兩個步驟形成一個原子操作

```c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

`bget` 在離開 `bcache.lock` 的 critical section 之後再去取得 buffer 的 sleep-lock 是安全的，因為只要 `b->refcnt` 不為 0，就可以防止該 buffer 被重複用於其他磁碟區塊。 sleep-lock 用來保護對這個區塊內容的讀寫操作，而 `bcache.lock` 則用來保護「哪些磁碟區塊已被快取」的相關資訊

如果所有的 buffer 都在忙碌中，表示有太多 process 同時在執行檔案系統呼叫了； 這時 `bget` 會直接 panic。 比較溫和的處理方式是讓該 process 睡眠直到有 buffer 變得可用，但這樣會有導致 deadlock 的可能性

一旦 `bread`（若有需要）從磁碟讀取資料並回傳 buffer 給呼叫者，呼叫者就能獨佔這個 buffer，可以任意讀取或寫入裡面的資料。 如果呼叫者修改了 buffer，就必須呼叫 `bwrite` 將變更寫回磁碟，然後才能釋放 buffer。 `bwrite`（[kernel/bio.c:107](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L107)）會呼叫 `virtio_disk_rw` 與磁碟硬體溝通

```c
// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}
```

當呼叫者使用完一個 buffer 時，必須呼叫 `brelse` 來釋放它（`brelse` 這個名稱是 b-release 的縮寫，雖然不好懂，但很值得學，因為它來自 Unix，並在 BSD、Linux、Solaris 等系統中沿用至今）。 `brelse`（[kernel/bio.c:117](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L117)）會釋放這個 buffer 的 sleep-lock，並將它移動到 linked list 的最前面（[kernel/bio.c:128-133](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L128-L133)）。 這讓整個 list 會根據 buffer 最近被使用的時間進行排序：list 最前面是最近被釋放的 buffer，最尾端是最久未使用的

```c
// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```

`bget` 中的兩段迴圈就是利用了這種排序方式：在尋找既有 buffer 的那一段，最糟情況下會掃過整個 list，但若有良好的區域性參考性（locality），從 `bcache.head` 開始沿著 next 指標往後找，就能縮短搜尋時間。 而在找出可回收 buffer 的那段，則是從 list 的尾端開始，沿著 `prev` 指標反向找，選出最久未使用的 buffer

## 8.4 Logging layer

在檔案系統設計中，當機復原（crash recovery）是一個非常有趣的問題。 這個問題的來自於許多檔案系統的操作會包含多次對硬碟的寫入，如果在這些寫入只完成一部分時系統發生當機，那麼硬碟上的檔案系統可能會變成不一致的狀態。 例如，假設當機發生在執行 file truncation（將檔案長度設為 0 並釋放其內容區塊）的過程中。 根據磁碟寫入的順序不同，當機後可能會出現以下兩種情況之一：第一種是 inode 仍然指向某個實際上已被標記為「空閒」的區塊，第二種是某個區塊已被配置但卻沒有任何 inode 參考它

第二種情況相對比較無害； 但對於第一種，若某個 inode 仍然指向一個已被釋放的區塊，在重新開機後很可能會引發嚴重的問題。 因為在重新開機之後，kernel 可能會把該區塊分配給另一個檔案，結果就變成兩個不同的檔案意外地指向同一個區塊。 如果 xv6 有支援多個使用者，這樣的狀況甚至可能變成一個安全漏洞，因為原本的檔案擁有者就可以讀寫新的、屬於其他使用者的檔案資料

xv6 使用一種簡化的 logging 機制來解決檔案系統操作期間可能發生當機的問題。 xv6 的系統呼叫並不會直接修改硬碟上的檔案系統資料結構，而是先將所有預計要寫入的內容描述記錄到硬碟上的 log 中。 等到所有寫入動作都被記錄到 log 之後，系統呼叫會在磁碟上寫入一個特殊的 commit 記錄，表示 log 內包含了一筆完整的操作。 到了這個階段，系統呼叫才會把這些寫入實際套用到檔案系統的資料結構上。 當這些寫入完成之後，系統呼叫會刪除磁碟上的 log

如果系統發生當機並重新啟動，檔案系統的程式會在執行任何 process 之前先進行以下的復原程序。 若 log 被標記為包含一筆完整的操作，那麼復原程式就會將這筆操作所對應的寫入內容複製到對應的檔案系統資料結構中。 反之，如果 log 沒有被標記為包含完整操作，復原程式就會忽略這份 log。 最後，復原程式會將 log 從磁碟上清除

xv6 的 log 能解決檔案系統操作期間發生當機的問題的原因是，如果當機發生在操作完成 commit 之前，那麼磁碟上的 log 不會被標記為完成，復原程式就會忽略它，此時硬碟的狀態就像這筆操作根本沒開始過一樣。 如果當機發生在操作 commit 之後，那麼復原程式會重播這筆操作的所有寫入，儘管在當機之前這些寫入可能已經部分套用到其資料結構中了。 無論是哪種情況，這個 log 都讓檔案系統的操作在面對當機時具有原子性：在復原之後，要麼這筆操作的所有寫入都存在磁碟上，要麼一個寫入都沒有

## 8.5 Log design

log 儲存在一個已知的固定位置，這個位置由 superblock 指定。 log 的內容包含一個 header 區塊，後面接著一連串更新後的區塊副本（稱為「logged blocks」）。 header 區塊中會有一個 sector 編號的陣列，對應每個 logged block 各自的磁區編號，還會有一個 log block 數量的計數值

```c
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n;
  int block[LOGSIZE];
};

struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing.
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
struct log log;
```

::: tip  
這邊先把整個流程簡單紀錄一下，因為當初在看的時候真的看很久，這邊原文寫得不太好，對整個流程先有個認識，看後面的原文才比較看得懂，你也可以先跳過往後看，發現有問題再回來看一下這段

在 xv6 的日誌（log）實作裡，`struct logheader` 是寫入日誌區第一個磁區（header block）的資料結構，也會在記憶體中維持一份相同內容，並隨著交易（transaction）中的 `log_write()` 呼叫而更新。 換句話說，包含等等會看到的 `strcut log` 在內，兩者都只有一個全域的實例（`log` 與 `log.lh`）

一個 transaction 是一組「要麼全部成功，要麼全部不做」的磁碟操作組合。 這些操作在日誌中被一次性地記錄、提交（commit），保證就算在中途當機也不會只執行了一半、造成檔案系統損毀

一個 transaction 通常包含：

- 寫入 inode（例如檔案長度更新）
- 修改 bitmap（標記 block 是否被使用）
- 寫入實際的資料 block（檔案內容）

的操作，舉例來說：

```c
begin_op();
ilock(f->ip);
writei(f->ip, ...);  // 可能寫好幾個 block
iunlock(f->ip);
end_op(); // 若這是最後一個使用者，就觸發 commit()
```

整個 `begin_op()` 到 `end_op()` 就是一個 transaction。 在全域範圍內 transaction <span class = "yellow">一次只會有一筆</span>，它會使一個全域的鎖 `log.lock` 來管理。 當不同的 system call 同時呼叫 `begin_op` 時，如果符合其內部條件（後面會提），他就會將 `log.outstanding` 這個計數器加一，以表示它也參予了此次 `transaction`

而 `logheader.n` 是個計數器，用來記錄目前這筆 transaction 最後要寫多少個「硬碟上的資料區塊」； `logheader.block` 則是一個磁區編號的清單，用來記錄「這筆 transaction 修改了哪些資料區塊」，本身並不包含資料（你也可以看到它是 `int` 的陣列）。 真正的資料都放在 buffer cache 中（於 bio.c 中的 `bcache.buf`），而 log block 的資料也是存在 buffer cache 裡的，而且只有在 commit 時才會被建立

一般的系統呼叫要寫入 block 的時候，如上方的 `writei` 函式，其內都會呼叫 `log_write` 來將磁碟編號 `blockno` 填入 `logheader.block`，最後再等到呼叫 `end_op` 時再實際的去做這個寫操作，每當有 system call 呼叫對應的 `end_op`（和 `begin_op` 是一組的），`log.outstanding` 就會被減一

而當 `end_op` 內發現所有系統呼叫的寫入操作都被記錄下來後（`log.outstanding == 0`），換句話說就是最後一個呼叫 `end_op` 的人，會去呼叫 `commit` 這個函式。 也因此，當有多個系統呼叫的寫入操作正等著被執行時，`logheader.block` 內就會存著不同組系統呼叫的寫入操作所寫的 buffer

還有，從這裡你也可以發現，所謂的「一筆」transaction 其實是在「第一個 `begin_op()` 成功」那一刻開始，直到 `log.outstanding` 由 1 變 0 而觸發 `commit()` 後結束的。 如果同時有多個 system call 交錯執行，它們各自的 `begin_op … end_op` 區段都被包在同一筆 transaction 內，而不是「每個 system call 就是一筆 transaction」

最後，`commit` 這個函式是整個過程裡面最重要的函式，只會在這裡被呼叫，其主要做四件事

1. `write_log`：把 `logheader.block` 內紀錄的磁碟編號對應的「記憶體上的 buffer cache」裡面的內容，複製到「硬碟上的 log block」中：
     ```c
     struct buf *to = bread(log.dev, log.start+tail+1); // log block
     struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
     memmove(to->data, from->data, BSIZE);
     bwrite(to);  // write the log
     brelse(from);
     brelse(to);
     ```
2. `write_head`：把記憶體中的 `log.lh` 結構覆寫到硬碟上的日誌區（block 2，見圖 8.2）的第 0 個 sector
   - 此時硬碟上的日誌區就有了完整的「要寫的資料」與 `logheader` 的內容了
   - `logheader` 裡面記錄了這些硬碟上的日誌區內的 block 的內容等等要搬去哪裡
   - 到這裡 `commit` 的重點（防止當機）就完成了，此時如果當機了，由於硬碟上的日誌區都還存著這些資料要去哪，所以開機時就有辦法復原
3. `install_trans`：把硬碟日誌區中的那些 block 搬到正確的位置上
4. `log.lh.n = 0; write_head();`：將記憶體與硬碟上的 `logheader` 的資訊都清 0

整體流程大概就是這樣，從上面你也可以看到，log block 的內容其實就是其對應的 data block 的副本  
:::

若磁碟上的 header 區塊中的計數器欄位為 0，表示目前 log 中不存在任何的 transaction； 若為非零值，則表示 log 中已包含一筆完整且已 commit 的 transaction，且其中有指定數量的 logged blocks。 xv6 會在 commit 時寫入這個 header 區塊（而不是事前），並會在把 logged blocks 寫入檔案系統後，將計數器清 0。 因此如果 transaction 執行到一半當機，log header 區塊中的計數器就會是 0； 如果在 commit 之後才當機，那麼計數器會是非零值

每個系統呼叫的程式碼都會標示出寫入序列的起點與終點，在這之間的寫入對 crash 來說必須是原子的。 在讓多個 process 同時執行檔案系統操作的情況下，log 系統允許將多個系統呼叫的寫入操作累積在同一個 transaction 中。 因此，單個 commit 可能包含了多個完整系統呼叫的寫入。 為了避免某個系統呼叫的寫入被拆分到不同的 transaction 中，log 系統只有在沒有任何檔案系統相關的系統呼叫在執行時才會進行 commit

將多筆 transaction 一起進行 commit 的概念被稱為「group commit」。 group commit 能減少磁碟操作的次數，因為它可以將 commit 的固定成本分攤到多筆操作上。 此外，group commit 也會讓磁碟系統一次接收到較多筆寫入操作，有機會讓磁碟在一次旋轉期間就完成所有寫入。 雖然 xv6 所用的 virtio driver 並不支援這種 batching，但 xv6 的檔案系統設計仍然允許這種做法的存在

xv6 在磁碟上預留了一塊固定大小的空間用來存放 log。 每次 transaction 中由系統呼叫寫入的所有區塊數量必須能夠塞進這塊 log 空間。 這會產生兩個後果：

1. 不能讓任何一個系統呼叫寫入超過 log 空間大小的區塊數  
   - 這對大多數系統呼叫來說不是問題，但有兩個系統呼叫可能會寫入大量區塊：`write` 與 `unlink`
     - 對於一個大型檔案的 `write` 操作，可能需要寫入很多個 data block、bitmap block，甚至還包含一個 inode block
     - `unlink` 一個大型檔案也可能會涉及多個 bitmap block 與 inode 的寫入
   - xv6 的 `write` 系統呼叫會將大型寫入拆分成多個較小的操作，每次都能夠塞進 log； 而 `unlink` 則沒造成實務上的問題，因為 xv6 的 file system 通常只會用到一個 bitmap block
2. 由於 log 空間有限，在 logging system 確認一個系統呼叫的所有寫入能夠完全塞入剩餘的 log 空間之前，不能允許該系統呼叫開始執行

## 8.6 Code: logging

一個 system call 中使用日誌系統的典型方式如下所示：

```c
begin_op();
...
bp = bread(...);
bp->data[...] = ...;
log_write(bp);
...
end_op();
```

`begin_op`（[kernel/log.c:127](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L127)）會等到日誌系統沒有在處理 commit，並且 log 中有足夠尚未被預留的空間，可以容納這次系統呼叫要寫入的資料後，才會繼續執行。 `log.outstanding` 會記錄目前有多少個系統呼叫預留了 log 空間，整體預留的空間大小為 `log.outstanding` 乘上 `MAXOPBLOCKS`。 將 `log.outstanding` 加一具有預留空間的功能，並能避免在這個系統呼叫的期間內觸發 commit。 程式碼保守地假設每次系統呼叫最多可能會寫入 `MAXOPBLOCKS` 個不同的 block

::: tip  
`begin_op()` 要做兩件事：

1. 確定可以塞得下
   - 空間分為兩種：
     - 已用空間 (`log.lh.n`)：這筆全域唯一的 transaction 已經記錄了多少「資料區塊」的數量
     - 預留空間：這是等等會用到的 log block，`log.outstanding` 代表有幾個 log block
2. 確定現在沒有 commit 在跑  
    因為 commit 會讀寫同一批 log block，若程序 A 在 commit、程序 B 又想 append 新 block，磁碟內容可能衝突

只有兩條件都成立，`begin_op()` 才會做 `log.outstanding++`，正式「卡位」成功，並持有這把額度直到 `end_op()`

至於對 commit 有「互斥」效果，是因為 commit 只會在 `log.outstanding` 變為 0 時觸發（見底下的 `end_op()`）。 任何其他的 system call 只要先 `begin_op()` 成功，`log.outstanding` 就大於 0，這期間 commit 一定不會開始。 也因此同一個 system call 內的多個 `log_write()` 肯定屬於同一筆 transaction，不怕被拆開  
:::

```c
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```

xv6 中的 `log_write`（[kernel/log.c:215](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L215)）會用來作為 `bwrite` 的代理（proxy）。 它會在記憶體中記錄該 block 的磁區編號，並為其在磁碟上的 log 中預留一個位置，同時會將這個 buffer 釘在 block cache 中，避免被 cache 排除。 這個 block 必須一直預留在 cache 中直到 commit 為止：在此之前，cache 中的資料是唯一的修改紀錄，在 commit 前不能把它寫回磁碟原本的位置，而且同一 transaction 中的其他讀取也必須能看到這些修改

`log_write` 會偵測在同一 transaction 中是否多次寫入了同一個 block，並讓該 block 重複使用同一個 log slot，這項最佳化被稱為吸收（absorption）。 這種情況很常見，例如：儲存多個檔案的 inode 的磁碟區在同一個 transaction 裡可能會被寫入好幾次。 透過將多次的磁碟寫入合併成一次，檔案系統可以節省 log 空間，並提升效能，因為最後只會有一個磁碟區的副本需要被寫入磁碟

```c
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  acquire(&log.lock);
  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorption
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```

`end_op`（[kernel/log.c:147](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L147)） 會先將 `log.outstanding` 減一。 如果計數器變成零，代表這是目前 transaction 中的最後一個系統呼叫，因此其會呼叫 `commit()` 來提交這筆 transaction：

```c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

整個 commit 的過程包含分為階段：

1. `write_log()`（[kernel/log.c:179](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L179)）會把該 transaction 中被修改過的 block，從 buffer cache 複製到磁碟上與其對應的 slot
2. `write_head()`（[kernel/log.c:103](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L103)）會把 log 的 header block 寫入磁碟：這就是 commit 點，如果系統在這步之後當機，恢復程序會根據 log 重播這次 transaction 的所有寫入
3. `install_trans`（[kernel/log.c:69](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L69)）會從 log 中讀出每個 block，並將其寫入檔案系統中正確的位置
4. `end_op` 會將 log header 中的計數器清 0，這動作必須在下一筆 transaction 開始寫入 logged block 之前完成，否則若系統當機，會導致恢復時錯誤地將「前一筆 transaction 的 header」與「後一筆 transaction 的 logged block」搭配使用

```c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```

`initlog`（[kernel/log.c:55](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L55)）會呼叫 `recover_from_log`（[kernel/log.c:117](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/log.c#L117)），而 `initlog` 則是在開機階段由 `fsinit` 呼叫的（[kernel/fs.c:42](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/fs.c#L42)），這會發生在第一個使用者行程啟動前（[kernel/proc.c:535](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L535)）。 它會讀取 log header，若 header 顯示 log 中包含一筆已提交的 transaction，`recover_from_log` 就會模擬 `end_op` 的動作來進行還原

```c
static void
recover_from_log(void)
{
  read_head();
  install_trans(1); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}
```

在 `filewrite`（[kernel/file.c:135](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/file.c#L135)）中可以看到一個使用 log 的例子，其 transaction 如下：

```c
begin_op();
ilock(f->ip);
r = writei(f->ip, ...);
iunlock(f->ip);
end_op();
```

這段程式碼被包在一個迴圈中，會將大型的寫入操作分成多個僅寫入少數磁區的小型 transaction，以避免 log 空間爆滿。 `writei` 呼叫的過程中會寫入許多不同的 block，包括該檔案的 inode、一個或多個 bitmap block，以及一些 data block
