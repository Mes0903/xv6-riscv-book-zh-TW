---
title: xv6 riscv book chapter 9：Concurrency revisited
date: 2025-08-04
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---

# xv6 riscv book chapter 9：Concurrency revisited

在核心設計中，要同時擁有良好的平行效能、並發下的正確性，以及可理解的程式碼，是一項重大挑戰。 直接使用 lock 是達成正確性最可靠的方式，但這並非總是可行的。 本章會列出一些例子：有些情況 xv6 被迫以複雜的方式使用 lock，也有些情況則採用了類似 lock 的技巧，但實際上沒有用 lock

## 9.1 Locking patterns

對於 cache 項目的鎖定常常是一項難題。 例如，檔案系統的 block cache（[kernel/bio.c:26](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L26)）會儲存最多 `NBUF` 個 disk block 的副本。 系統必須確保同一個 disk block 在 cache 中最多只能有一份副本，否則不同的 process 可能會對同一個 block 的不同副本進行互相衝突的修改。 每個被 cache 的 block 都儲存在一個 `struct buf`（[kernel/buf.h:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/buf.h#L1)）中。 `struct buf` 本身有一個 lock 欄位，用來確保同一時間只有一個 process 能使用該 block

然而，這樣的 lock 並不夠：假如某個 block 還不在 cache 中，且有兩個 process 同時想使用它，這時因為還沒有對應的 `struct buf` 存在，所以根本沒東西可鎖。 xv6 透過一個額外的 lock（`bcache.lock`）來保護「目前已被 cache 的 block 的集合」

任何需要查找某個 block 是否已被 cache（例如 `bget`, [kernel/bio.c:59](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/bio.c#L59)），或修改 cache 集合的程式，都必須持有 `bcache.lock`。 一旦找到了所需的 block 與其對應的 `struct buf`，即可釋放 `bcache.lock`，改為鎖住該 block 對應的 `struct buf`。 這是一個常見的模式：一把鎖保護整個集合，另一把鎖保護集合中的個別項目

一般來說，取得 lock 的函式會在同一個函式中釋放該 lock。 但更精確的說法是：lock 會在一段需要表現為原子操作的序列開始時被取得，而在這段序列結束時釋放。 如果這段序列的開始與結束發生在不同的函式、不同的 thread，甚至不同的 CPU 上，那麼 lock 的取得與釋放也必須跨越這些邊界。 lock 的目的是阻止其他使用者介入，而不是將資料綁定在某個特定 agent 上

一個例子是 `yield` 中的 `acquire`（[kernel/proc.c:512](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L512)），其對應的釋放發生在 scheduler thread，而非原本呼叫者所在的 process。 另一個例子是 `ilock` 中的 `acquiresleep`（[kernel/fs.c:293](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/fs.c#L293)），該程式在讀取磁碟時可能會 sleep，而醒來時可能已經在不同的 CPU 上了，因此 `acquire` 與 `release` 可能會發生於不同 CPU

釋放內部含有 lock 的物件的操作會非常敏感，因為單純持有該 lock 並不足以保證釋放行為是正確的。 典型的問題是：當某個 thread 正在 `acquire` 該 lock 等待使用這個物件時，若此時另一個 thread 將該物件釋放，那麼這樣也會隱含地釋放掉 lock 本身，導致正在等待的 thread 發生錯誤

對此的一種解法是追蹤這個物件的引用計數（reference count），只有當最後一個引用消失時才釋放該物件。 請參見 `pipeclose`（[kernel/pipe.c:59](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/pipe.c#L59)）作為範例； 其中 `pi->readopen` 與 `pi->writeopen` 用來追蹤是否仍有 file descriptor 指向該 pipe

```c
void
pipeclose(struct pipe *pi, int writable)
{
  acquire(&pi->lock);
  if(writable){
    pi->writeopen = 0;
    wakeup(&pi->nread);
  } else {
    pi->readopen = 0;
    wakeup(&pi->nwrite);
  }
  if(pi->readopen == 0 && pi->writeopen == 0){
    release(&pi->lock);
    kfree((char*)pi);
  } else
    release(&pi->lock);
}
```

通常我們會在一連串針對一組相關資料的讀寫操作的外圍加上 lock，這樣可以保證其他 thread 在使用時，只會看到已完整更新的資料（只要它們自己也有加鎖）。 那麼如果只需要寫入一個共享變數該怎麼辦？ 例如 `setkilled` 和 `killed`（[kernel/proc.c:619](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L619)）在操作 `p->killed` 這個變數時就有使用 lock。 若沒有加鎖，可能會發生一個 thread 寫入 `p->killed` 的同時，另一個 thread 正在讀取它，這就構成一個 race

根據 C 語言的標準，這類 race 屬於 undefined behavior，也就是該程式可能會當機或產生錯誤結果（參見 cppreference 上的「[Threads and data races](https://en.cppreference.com/w/c/language/memory_model.html)」）。 加上 lock 就能避免這樣的 race，也就避免了 undefined behavior

race 會破壞程式正確性的其中一個原因是，如果沒有使用 lock 或其他等效機制，編譯器可能會產生與原始 C 程式碼邏輯完全不同的機器碼來進行記憶體的讀寫。 例如，一個 thread 呼叫 `killed` 時，其機器碼可能會將 `p->killed` 的值複製到暫存器中，之後就只讀這個快取的值； 這將導致該 thread 永遠看不到其他 thread 對 `p->killed` 所做的任何寫入。 而使用 lock 則能避免這種快取行為

## 9.2 Lock-like patterns

在 xv6 的許多地方，系統會以 reference count 或 flag 的方式，模擬類似 lock 的行為，用來表示某個物件目前處於已配置（allocated）狀態，因此不應該被釋放或重新使用。 像是 process 的 `p->state` 就具有這種效果，而 `file`、`inode` 與 `buf` 結構中的 reference count 也是如此。 雖然這些 flag 或 reference count 本身都會受到 lock 保護，但真正防止物件被過早釋放的，其實是這些 reference count 本身

檔案系統會使用 `struct inode` 的 reference count 作為一種共享 lock，這樣可以讓多個 process 同時持有，以避免那些使用一般 lock 可能發生的 deadlock。 例如，`namex` 中的迴圈（[kernel/fs.c:652](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/fs.c#L652)）會依序對每個 path component 所對應的目錄加鎖

而 `namex` 在每次迴圈結束時必須釋放該鎖，否則若 path 中包含 `.`（例如 `a/./b`），就可能會導致自身 deadlock。 也可能與另一個正在查找 `..` 的 thread 發生死結。 如第八章中解釋的，解法是讓迴圈將目前的 directory inode 帶到下一輪，但只增加 reference count 而不加鎖

有些資料項目在不同時期會受到不同機制的保護，有時甚至並非透過明確的 lock，而是透過 xv6 程式碼的結構隱含地避免了並發存取。 例如，當一個 physical page 處於 free 狀態時，它會受到 `kmem.lock` 的保護（[kernel/kalloc.c:24](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kalloc.c#L24)）。 若該 page 被分配成一個 pipe（[kernel/pipe.c:23](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/pipe.c#L23)），它則會受到另一把 lock（`pi->lock`）的保護。 如果這個 page 被分配為某個新 process 的 user memory，那它甚至完全沒有受到任何 lock 保護，而是仰賴配置器的行為保證：只要 page 尚未被釋放，就不會分配給其他 process，因此也不會有並發使用的問題

一個新 process 的 memory 擁有權的轉移過程相當複雜：一開始由 parent 在 `fork` 中配置與操作，然後由 child 使用，最後當 child 結束時，又回到 parent 手上，並交由 `kfree` 釋放。 這裡可學到兩點：第一，某個資料物件在生命週期的不同階段，可能會以不同方式被保護； 第二，這種保護有時可能是隱含的結構性設計，而非顯式地使用 lock

最後一個類似 lock 的例子是：在呼叫 `mycpu()`（[kernel/proc.c:83](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L83)）時需要關閉中斷。 關閉中斷可以讓該段程式在執行期間，不會被 timer interrupt 打斷，進而強制發生 context switch，將 process 切換到另一顆 CPU
