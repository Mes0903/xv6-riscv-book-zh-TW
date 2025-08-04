---
title: xv6 riscv book chapter 3：Page tables
date: 2025-07-30
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---

# xv6 riscv book chapter 3：Page tables

Page table 是作業系統用來為每個行程提供私有位址空間與記憶體的最常見的機制。 page table 決定了記憶體位址的含義，以及哪些物理記憶體區段可以被存取。 它們讓 xv6 能夠隔離不同行程的位址空間，並將它們多工使用在單一的物理記憶體上

page table 之所以被廣泛使用，是因為它們提供了一層間接性，使作業系統能夠應用許多「技巧」。 xv6 就使用了一些技巧：像是將相同的記憶體（例如 trampoline page）映射到多個位址空間中，並且用未映射的 page 來保護 kernel 與 user 的 stack。 接下來的章節會說明 RISC-V 硬體所提供的 page table 功能，以及 xv6 是如何使用這些功能的

## 3.1 Paging hardware

回顧一下，RISC-V 的指令（無論是使用者或 kernel ）所操作的是虛擬位址。 機器的 RAM，也就是實體記憶體，則是以實體位址來索引。 RISC-V 的 page table 硬體會將這兩種位址連接起來，將每個虛擬位址對應到一個實體位址來完成映射

xv6 運行在 Sv39 的 RISC-V 架構上，這代表 64 位元虛擬位址中，只有最低的 39 個位元會被使用，最上面的 25 個位元則不會被使用。 在 Sv39 的設定下，一個 RISC-V page table 在邏輯上是一個包含 2<sup>27</sup>（134,217,728）個 PTE 的陣列。 每個 PTE 包含一個 44 位元的 page frame 編號（PPN）以及一些旗標

::: tip  
page frame 指的是實體記憶體的 page，原文為 physical page，因此 page frame 的編號才會縮寫為 PPN（physical page number）  
:::

paging hardware 會使用 39 位元中最高的 27 位元作為索引查找 page table，找到對應的 PTE，然後組合成一個 56 位元的實體位址：位址的高 44 位元來自 PTE 裡的 PPN，低 12 位元則直接複製自原本虛擬位址中的低 12 位。 圖 3.1 顯示了這個流程，其使用一個簡化為 PTE 陣列的邏輯 page table 來呈現（更完整的結構請見圖 3.2）。 page table 讓作業系統可以用 4096（2<sup>12</sup>）位元組對齊的區塊為單位，控制虛擬位址到實體位址的對應關係。 這種區塊就被稱作「page」

![（Figure 3.1: RISC-V virtual and physical addresses, with a simplified logical page table.）](image/riscv_address.png)

在 Sv39 的 RISC-V 架構中，虛擬位址的高 25 位並不會參與轉換。 而在實體位址方面也預留了成長的空間：在 PTE 的格式中，PPN 還可以再增加 10 個位元。 RISC-V 的設計者是根據技術的發展預測來選定這些數值的，2<sup>39</sup> 位元組等於 512GB，對於在 RISC-V 電腦上運行的應用程式來說應該已經足夠。 2<sup>56</sup> 則提供了足夠的實體記憶體空間，在可見的未來能容納許多 I/O 裝置與 RAM 模組。 如果未來還需要更多，RISC-V 的設計者也已定義了擁有 48 位元的虛擬位址空間的 Sv48<sup>[[1]](#1)</sup>

如圖 3.2 所示，RISC-V 的 CPU 會透過三個步驟將虛擬位址轉換為實體位址。 page table 在實體記憶體中會以三層的樹的形式儲存，這棵樹的根是一個 4096 位元組的 page table，裡面包含 512 個 PTE，這些 PTE 內也各都儲存著下一層 page table 的實體位址。 而該 page table 中的每個 PTE 所指向的 page table，其內會包含 512 個最底層的 PTE。 每個 page table 都使用了一個 page 的大小（4096 位元組）來儲存

paging hardware 會使用 27 位元中最高的 9 位來在 root page table 中選取一個 PTE，中間的 9 位元用來在下一層的 page table 中選取一個 PTE，而最底下的 9 位元則用來選擇最終的 PTE（在 Sv48 的 RISC-V 中，page table 有四層，虛擬位址中的第 39 到 47 位會用來索引最頂層的 page table）

![（Figure 3.2: RISC-V address translation details.）](image/riscv_pagetable.png)

如果在位址轉換過程中所需的三個 PTE 中有任何一個不存在，paging hardware 就會產生一個「page-fault 例外」，並交由 kernel 來處理這個例外（詳見第四章）

相較於圖 3.1 的單層設計，圖 3.2 所示的三層結構提供了一種更節省記憶體的方式來記錄 PTE。 在許多虛擬位址範圍根本沒有被映射的情況下，三層結構有機會能直接省略多個 page table。 例如，如果一個應用程式只使用從位址 0 開始的幾個 page，那麼第一層 page table 內的第 1 到 511 的 PTE 都會是無效的，kernel 不必耗費 page 來存這 511 個第二層 page table，也不需要分配這 511 個第二層 page table 所對應到的底層 page table。 因此，在這個例子中，三層結構可以節省 511 個 page 的第二層 page table，以及 511×512 個 page 的底層 page table 

雖然 CPU 會在執行 `load` 或 `store` 指令時，由硬體自動走訪三層結構，但三層結構有個潛在缺點是：CPU 必須從記憶體中載入三個 PTE 才能完成虛擬位址到實體位址的轉換。 為了減去從實體記憶體載入 PTE 的開銷，RISC-V 的 CPU 會將 PTE 快取在一個稱為 Translation Look-aside Buffer（TLB）的結構中

每個 PTE 都包含一些旗標位元，用來告訴 paging hardware 這個對應的虛擬位址允許被如何使用。 `PTE_V` 表示這個 PTE 是否存在：如果其沒被設置，則對該 page 的存取會引發例外。 `PTE_R` 決定指令能否讀取該 page。 `PTE_W` 決定能否寫入該 page。 `PTE_X` 決定 CPU 是否可以將該 page 的內容作為指令來執行。 `PTE_U` 決定 user mode 下的指令是否可以存取該 page； 如果沒設置 `PTE_U`，則僅能在 supervisor mode 中使用該 PTE。 圖 3.2 展示了這整個是如何運作的。 這些旗標以及其他與 page 硬體有關的結構都定義在 [kernel/riscv.h](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h) 中

若要讓 CPU 使用某個 page table，kernel 必須將 root page table 的 page 的實體位址寫入 `satp` 暫存器中，這樣接下來 CPU 執行的所有指令所產生的位址，都會使用 `satp` 指向的 page table 來進行轉換。 每顆 CPU 都有自己的 `satp` 暫存器，因此不同的 CPU 可以同時執行不同的行程，各自使用其私有的位址空間與 page table。 從 kernel 的角度來看，page table 就是儲存在記憶體中的資料結構，kernel 會使用類似操作其他樹狀資料結構的方式來建立與修改 page table 

這裡對書中所使用的一些術語做個簡要說明。 「實體記憶體」是指 RAM 中的儲存單元。 一個實體記憶體位元組會有一個稱為「實體位址」的位址。 那些會解參考位址的指令（例如 `load`、`store`、`jump`、function call）只會使用虛擬位址，這些虛擬位址會先由 paging hardware 轉換為實體位址，再送到 RAM 進行讀寫

「位址空間」是指在某個 page table 中有效的虛擬位址集合； xv6 中的每個行程都有自己的使用者位址空間，xv6 kernel 本身也有自己的位址空間。 「使用者記憶體」是行程的使用者位址空間加上 page table 允許該行程存取的實體記憶體。 「虛擬記憶體」是一組與 page table 管理有關的概念與技術，並透過它們來實現如隔離等目標

## 3.2 Kernel address space

xv6 為每個行程維護一個 page table，用來描述該行程的使用者位址空間，此外還有一份單獨、全域的 page table 描述 kernel 的位址空間。 kernel 會配置自己位址空間的佈局（layout），使其能夠在預期的虛擬位址上存取實體記憶體與各種硬體資源。 圖 3.3 顯示這個佈局如何將 kernel 虛擬位址對應到實體位址。 [kernel/memlayout.h](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/memlayout.h) 中宣告了 xv6 kernel 記憶體佈局的各種常數

![（Figure 3.3: On the left, xv6’s kernel address space. RWX refer to PTE read, write, and execute permissions. On the right, the RISC-V physical address space that xv6 expects to see.）](image/xv6_layout.png)

QEMU 模擬了一台電腦，其中的 RAM（實體記憶體）從實體位址 `0x80000000` 開始，持續到 `0x88000000` 以上，這段範圍在 xv6 中稱為 `PHYSTOP`。 QEMU 的模擬也包含像是硬碟介面這樣的 I/O 裝置，QEMU 以記憶體映射控制暫存器（memory-mapped control registers）的方式，將這些裝置的介面暴露給軟體，這些暫存器位於實體位址空間中小於 `0x80000000` 的位置。 kernel 可以透過讀寫這些特殊的實體位址與裝置互動，換句話說這些讀寫會與裝置硬體溝通，而非與 RAM 互動。 第四章會解釋 xv6 是如何與裝置互動的

kernel 透過「直接映射（direct mapping）」的方式來存取 RAM 與 memory-mapped 的裝置暫存器，其會將資源映射到與其實體位址相同的虛擬位址上（VA == PA），例如 kernel 本身在虛擬位址空間與實體記憶體中都位於 `KERNBASE=0x80000000`。 直接映射能簡化 kernel 對實體記憶體的讀寫程式碼，例如在 `fork` 配置子行程的使用者記憶體時，配置器會回傳那塊記憶體的實體位址； `fork` 在複製父行程的使用者記憶體到子行程時，會直接把這個實體位址當作虛擬位址使用

有一些 kernel 的虛擬位址並不是直接映射的：

- Trampoline page：  
  它被映射到虛擬位址空間的最頂部，而使用者的 page table 也會有這個相同的映射。 第四章會討論 trampoline page 的用途，但在這裡我們可以看到一個有趣的 page table 用法：一個 page frame（存放 trampoline 程式碼）在 kernel 的虛擬位址空間中被映射了兩次，一次在虛擬空間頂部，另一次則為直接映射
- kernel stack page：  
  每個行程都有自己的 kernel stack，它會被映射到較高的虛擬位址位置，而 xv6 會在其下方留下一個沒有被映射的「guard page」。 這個 guard page 的 PTE 是無效的（也就是 `PTE_V` 沒有設置），這樣當 kernel stack 溢出時，通常就會觸發例外並使 kernel 發生 panic。 若沒有 guard page，stack 溢出就可能會覆蓋其他 kernel 記憶體，導致錯誤行為，而比起默默地發生錯誤執行，有出錯、崩潰是比較可以接受的

雖然 kernel 透過高位址的映射使用它的 stack，但 kernel 其實也可以透過直接映射的位址存取這些 stack。 另一種設計可能會只使用直接映射的方式，直接在那個位址操作 stack。 不過在這種設計中，如果要提供 guard page，就得取消某些本來會對應到實體記憶體的虛擬位址，這會讓記憶體變得難以使用

kernel 將 trampoline page 與 kernel 程式碼的 page 設置為具有 `PTE_R` 與 `PTE_X` 的權限，這表示 kernel 可以在這些 page 上讀取並執行指令。 其他 page 則被設置為具有 `PTE_R` 與 `PTE_W` 的權限，以便 kernel 能夠對這些 page 進行讀寫。 至於 guard page，則被設為無效映射

::: tip  
kernel 一開始使用的是 Bare Mode（`satp.MODE == 0`）：

```c
// entry.S jumps here in machine mode on stack0.
void
start()
{
  ...
  // disable paging for now.
  w_satp(0);
  ...
}
```

但「kernel 透過 direct mapping 的方式來存取 RAM 與 memory-mapped 的裝置暫存器」這句話，並不是在指 Bare mode。 它說的是在初始化 kernel page table 的時候，他會「手動」依照 Sv39 的格式，將 VA 映射到與其相同位址的 PA。 因此在後面已啟用 Sv39 的環境下，你把拿到的 VA 以 Sv39 的規則去查 kernel page table 時，最後得出的 PA 還是會剛好等於 VA（除了之前提到的 trampoline page 之類的）

以 uart register 為例，他在 kernel page table 中被這麼初始化：

```c
// Make a direct-map page table for the kernel.
pagetable_t
kvmmake(void)
{

  pagetable_t kpgtbl;

  kpgtbl = (pagetable_t) kalloc();
  memset(kpgtbl, 0, PGSIZE);

  // uart registers
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);
  ...
}
```

其中 `UART0 = 0x10000000L`，`PGSIZE = 4096`。 而如前面所述 `kvmmap` 會呼叫 `mappages`：

```c
// add a mapping to the kernel page table.
// only used when booting.
// does not flush TLB or enable paging.
void
kvmmap(pagetable_t kpgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(kpgtbl, va, sz, pa, perm) != 0)
    panic("kvmmap");
}
```

因此你可以看見，在其傳入 `kvmmap` 的參數中，`va` 與 `pa` 是直接寫了相同的值 `UART0`，這就是 direct mapping 的意思。 上方是 mmap 裝置的初始化，而對於 RAM 也是：

```c
// map kernel text executable and read-only.
kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

// map kernel data and the physical RAM we'll make use of.
kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
```

這兩行就把整個 Physical memory 都包含進來了（`KERNBASE` 至 `PHYSTOP`，見圖 3.3）。 而 `kvmmap` 內的 `mappages` 會利用 `walk` 來判斷你給的參數 `va` 在 kernel page table 中是否已經被映射了：

```c
// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa.
// va and size MUST be page-aligned.
// Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("mappages: va not aligned");

  if((size % PGSIZE) != 0)
    panic("mappages: size not aligned");

  if(size == 0)
    panic("mappages: size");
  
  a = va;
  last = va + size - PGSIZE;
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("mappages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

`walk` 固定會走訪三層 page table，如果途中發現某個 PTE 的值還是 0（未配置），就會用 `kalloc` 要一個 page frame，並把該 PTE 指向它：

```c
// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

也因此 kernel page table 一樣有三層，才可以依照 Sv39 的格式去查表。  再來對於 trampoline page 和 per-process 的 kernel stack，他又另外做了一次映射，但卻不是以 direct mapping 的方式：

```c
// map the trampoline for trap entry/exit to
// the highest virtual address in the kernel.
kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

// allocate and map a kernel stack for each process.
proc_mapstacks(kpgtbl);
```

因此上面才說這兩個東西可以用高位的虛擬位址來訪問，也可以走 direct mapping 的路線  
:::

## 3.3 Code: creating an address space

xv6 中大多數負責操作位址空間與 page table 的程式碼都寫在 vm.c（[kernel/vm.c:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L1)）中。 主要的資料結構是 `pagetable_t`，它實際上是一個指向 RISC-V root page table 的 page 的指標。 `pagetable_t` 的實例可能是 kernel 的 page table，也可能是某個行程的 page table。 相關的主要函式有 `walk` 與 `mappages`，前者用來找出某個虛擬位址對應的 PTE，後者會為新的映射關係建立對應的 PTE

以 `kvm` 開頭的函式會操作 kernel 的 page table； 以 `uvm` 開頭的函式會操作 user 的 page table； 其他函式則可能同時用於兩者。 `copyout` 與 `copyin` 用來從系統呼叫的引數提供的使用者虛擬位址中複製資料進出，這兩個函式之所以寫在 vm.c 裡，是因為它們必須顯式地將虛擬位址轉換成對應的物理位址

在開機流程的早期，`main` 會呼叫 `kvminit`，透過 `kvmmake` 建立 kernel 的 page table。 這個呼叫發生在 xv6 尚未啟用 RISC-V 的 paging 功能之前，因此當時的位址仍直接對應到實體記憶體。 `kvmmake` 會先分配一個 page 的實體記憶體作為 root page table，接著呼叫 `kvmmap` 來設置 kernel 所需的映射關係。 這些映射包含了 kernel 的程式與資料、本機到 `PHYSTOP` 為止的實體記憶體，以及實際上是裝置的某些記憶體區段。 `proc_mapstacks` 為每個行程配置一個 kernel stack，它會呼叫 `kvmmap`，把每個 stack 映射到由 `KSTACK` 產生的虛擬位址，同時為無效的 guard page 預留空間

`kvmmap`（[kernel/vm.c:132](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L132)）會呼叫 `mappages`（[kernel/vm.c:144](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L144)），針對目標範圍內的每個虛擬位址，以 page 大小為間格，將其映射關係加入到 page table 中。 對於每個要映射的虛擬位址，`mappages` 會呼叫 `walk` 找到該位址對應的 PTE，然後初始化這個 PTE，填入對應的 PPN、所需的存取權限（例如 `PTE_W`、`PTE_X` 或 `PTE_R`），並設置 `PTE_V` 將該 PTE 標記為有效 page 

`walk`（[kernel/vm.c:86](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L86)）模擬 RISC-V paging hardware 的行為，用來查找某個虛擬位址對應的 PTE。 `walk` 一次會往下走訪一層 page table，並使用該層虛擬位址的 9 個位元來索引對應的 page table。 在每一層 page table 當中，它可能會找到下一層 page table 的 PTE，或者是最終 page 的 PTE（[kernel/vm.c:92](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L92)）。 如果第一層或第二層的 page table 中的 PTE 無效，表示該層的 page 尚未配置； 如果設置了 `alloc` 引數，`walk` 就會為新 page table 配置一個新的 page，並把它的實體位址寫入該 PTE。 最終 `walk` 會回傳樹中最底層那個 PTE 的位址（[kernel/vm.c:102](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L102)）

上述的程式碼只能在實體記憶體已被直接映射到 kernel 的虛擬位址空間內的情況下執行。 例如，當 `walk` 向下走訪 page table 時，它會從某個 PTE 中取得下一層 page table 的實體位址（[kernel/vm.c:94](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L94)），然後把這個位址當作虛擬位址使用，來存取下一層的 PTE（[kernel/vm.c:92](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L92)）

`main` 會呼叫 `kvminithart`（[kernel/vm.c:62](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L62)）來載入 kernel 的 page table，這個函式會將 root page table 的 page 的實體位址寫入暫存器 `satp`，之後 CPU 就會開始使用這份 kernel 的 page table 來進行位址轉譯。 由於 kernel 使用的是直接映射，接下來的指令所使用的虛擬位址將會正確地映射到對應的實體記憶體位址上

每顆 RISC-V CPU 都會將 PTE 快取在 TLB（Translation Look-aside Buffer）中，而當 xv6 修改 page table 時，它必須通知 CPU 將對應的 TLB 快取項目作廢。 否則之後 TLB 可能會使用到過時的快取映射，進而指向一個已經被分配給其他行程的 page frame，導致某個行程不小心寫入其他行程的記憶體

RISC-V 提供一條名為 `sfence.vma` 的指令，用於清空當前 CPU 的 TLB。 xv6 會在 `kvminithart` 中重新載入 `satp` 後執行 `sfence.vma`，或在切換至使用者 page table 的 trampoline 程式碼中，於返回 user space 之前執行 `sfence.vma`。 在更改 `satp` 之前也必須執行一次 `sfence.vma`，以等待所有的 load 與 store 操作完成，這能確保先前對 page table 的更新已完成，並且也能保證先前的 load 與 store 操作會使用舊的 page table，而不是新的 page table 

::: tip  
`sfence.vma` 不只是用來清除 TLB，也可以作為一種記憶體屏障（memory barrier），確保舊 page table 的操作完成後，才開始使用新 page table，以避免順序錯亂造成的錯誤  
:::

為了避免整個 TLB 被清空，RISC-V CPU 可能會支援 ASID<sup>[[1]](#1)</sup>。 這樣 kernel 就可以只清除屬於特定地址空間的 TLB 項目。 但 xv6 並未使用這項功能

## 3.4 Physical memory allocation

kernel 在執行期間必須為 page table、使用者記憶體、kernel stack，以及 pipe 緩衝區分配與釋放實體記憶體。 xv6 使用從 kernel 結束位址到 `PHYSTOP` 之間的實體記憶體區域作為執行期間的配置來源，每次以 4096 位元組為單位配置與釋放整個 page。 它透過將這些 page 本身串成一個 linked list 來追蹤 free page，配置時會從 list 中取出一個 page，而釋放時則是將該 page 加入 list 中

## 3.5 Code: Physical memory allocator

記憶體配置器實作於 kalloc.c（[kernel/kalloc.c:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kalloc.c#L1)）中。 這個配置器是一個可分配的實體記憶體 page 所組成的「free list」，其的元素為 `struct run`，對應到一個 free page 

因為這些 free page 內並沒存其他東西，因此配置器會把每個 free page 對應的 `run` 結構體直接存在該 page 裡面，使配置器之後能夠取得這個 free list 的記憶體。 這個 free list 還受到一個自旋鎖的保護（[kernel/kalloc.c:21-24](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kalloc.c#L21-L24)），它們會一起被包在一個結構體裡，以明確表示該鎖保護的是此結構體內的欄位。 目前可以先忽略鎖以及 `acquire` 和 `release` 的呼叫，第六章會詳細討論 locking

`main` 函式會呼叫 `kinit` 來初始化配置器（[kernel/kalloc.c:27](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kalloc.c#L27)），其會將 free list 初始化為包含「從 kernel 結尾到 `PHYSTOP` 之間」的所有 page。 理論上 xv6 應該要透過解析硬體所提供的設定資訊來判斷可用的實體記憶體大小，但 xv6 採取了簡化的做法：直接假設機器擁有 128MB 的記憶體。 `kinit` 會呼叫 `freerange`，並對每一個 page 都呼叫 `kfree`，以將記憶體加入 free list

由於 PTE 只能對齊到 4096 位元組（即 4096 的倍數）的實體位址，因此 `freerange` 使用 `PGROUNDUP` 來確保只會釋放有對齊的實體位址。 配置器一開始沒有任何可用的記憶體，這些 `kfree` 的呼叫則為它提供了可以管理的記憶體

配置器有時會將位址當作整數使用，以便對它們進行數學運算（例如在 `freerange` 中走訪所有 page），有時又會將位址當作指標使用，用來讀寫記憶體（例如操作儲存在各 page 中的 `run` 結構）； 這種「位址的雙重用途」是配置器的實作中充滿 C type cast 的主要原因

`kfree`（[kernel/kalloc.c:47](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kalloc.c#L47)）會先將要釋放的記憶體中的每個位元組都設為數值 1。 這樣一來，若有程式在釋放後仍使用該記憶體（也就是所謂的「懸空參考（dangling reference）」），它讀取到的也會是雜訊資料而不是原本的正確內容，理論上可以更快地暴露錯誤。 接下來，`kfree` 會將該 page 加入 free list 的前端：它將實體位址（`pa`）轉型為指向 `struct run` 的指標，將原本 free list 的開頭記錄在 `r->next`，然後再將 free list 的開頭設為 `r`。 而 `kalloc` 則會從 free list 中取出（removes）並回傳第一個元素

## 3.6 Process address space

每個行程都有自己的 page table，而當 xv6 在行程間切換時，也會隨之切換 page table。 圖 3.4 比圖 2.3 更詳細地展示了一個行程的位址空間。 行程的使用者記憶體從虛擬位址 0 開始，可以一直成長到 `MAXVA`（[kernel/riscv.h:379](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L379)），這使得一個行程理論上能夠存取高達 256 GB 的記憶體

![（Figure 3.4: A process’s user address space, with its initial stack.）](image/processlayout.png)

一個行程的位址空間由多個 page 組成，這些 page 包括：儲存程式碼的 page（xv6 為其設定的權限為 `PTE_R`、`PTE_X` 和 `PTE_U`）、包含預先初始化資料的 page、一個用作 stack 的 page，以及數個用作 heap 的 page。 xv6 為資料、stack 與 heap 對應的 page 所設定的權限為 `PTE_R`、`PTE_W` 和 `PTE_U`

在使用者位址空間中設定權限，是強化使用者行程安全性的一種常見技巧。 如果 text 段被映射為具有 `PTE_W` 權限的 page，那麼行程就可能會不小心修改到自己的程式碼； 例如，若有程式錯誤導致對空指標寫入，就可能會改寫位於位址 0 的指令，接著程式繼續執行，造成更嚴重的後果

為了立即偵測這類錯誤，xv6 在映射 text 段時不會給予 `PTE_W` 權限，因此如果程式誤寫入位址 0，硬體將會拒絕這次寫入並產生 page 錯誤，接著 kernel 會終止該行程並輸出一條錯誤訊息，幫助開發者追蹤問題。 同樣地，透過不為 data 段映射到的 page 設置 `PTE_X` 權限，使用者程式便無法意外跳躍到 data 段的位址，並從那裡開始執行

在現實世界中，透過精確地設定權限來強化行程的安全性，也有助於防禦各種安全攻擊。 攻擊者可能會為某些程式（例如一個網頁伺服器）設計一些精巧的輸入，藉此觸發程式中的某個錯誤，並進一步將其變成可被利用的漏洞。 謹慎地設定權限，加上其他技術（例如隨機化使用者位址空間的配置），能有效增加此類攻擊的難度

stack 段僅佔用一個 page，圖 3.4 中顯示的是由 `exec` 建立的初始內容。 命令列引數的字串，以及指向這些字串的指標陣列，會被放在 stack 的最頂部。 緊接著在它們之下，是一些讓程式可以從 `main` 開始執行的資料，就像是呼叫了 `main(argc, argv)` 一樣

為了偵測 user stack 溢出分配範圍的情況，透過清除 page 的 `PTE_U` 標誌，xv6 在 stack 下方放置了一個無法存取的「guard page」。 若 user stack 溢出並試圖使用 stack 下方的位址，因為該 guard page 對 user mode 的程式是不可存取的，硬體將產生 page 錯誤例外。 現實中的作業系統也有可能會選擇在 stack 溢出時自動配置更多記憶體

當某個行程向 xv6 索要更多使用者記憶體時，xv6 會擴展該行程的 heap 段。 首先會使用 `kalloc` 配置 page frame，然後在該行程的 page table 中新增指向這些 page frame 的 PTE，並為這些 PTE 中設置 `PTE_W`、`PTE_R`、`PTE_U` 和 `PTE_V` 標誌。 大多數行程並不會使用整個使用者位址空間，對於未使用的 `PTE`，xv6 會清除其 `PTE_V` 

這裡我們看到了 page table 運用的幾個典型範例。 首先，不同行程的 page table 會將使用者位址映射到不同的實體記憶體 page，因此每個行程擁有各自私有的使用者記憶體。 其次，每個行程都會看到自己的記憶體是個從 0 開始且連續排列的虛擬位址空間，而實體記憶體則可以是不連續的。 第三，kernel 會在使用者位址空間頂端映射一個包含 trampoline 程式碼的 page（不設置 `PTE_U`），因此這個單一 page frame 會出現在所有行程的位址空間中，但只有 kernel 可以使用它

## 3.7 Code: sbrk

`sbrk` 是一個系統呼叫，用來讓一個行程可以擴增或縮減它的記憶體空間，這個系統呼叫是由 `growproc`（[kernel/vm.c:233](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L260)）函式實作的。 `growproc` 會根據其引數 `n` 的正負，來呼叫 `uvmalloc` 或 `uvmdealloc`。 `uvmalloc` 會使用 `kalloc` 配置實體記憶體，並將分配到的記憶體清 0，再透過 `mappages` 將各 PTE 加入使用者的 page table。 `uvmdealloc` 則會呼叫 `uvmunmap`，該函式會使用 `walk` 尋找 PTE，並透過 `kfree` 釋放它們所對應的實體記憶體

xv6 為每個行程都建了 page table，不僅僅是為了告訴硬體如何將使用者虛擬位址映射到實體記憶體，同時也作為唯一的記錄，指出哪些實體記憶體 page 被分配給了該行程。 這就是為什麼在釋放使用者記憶體時（如在 `uvmunmap` 中），必須檢查該行程的 page table 

## 3.8 Code: exec

`exec` 是一個系統呼叫，會將某個行程的使用者位址空間替換為從檔案中讀取的資料，這個檔案被稱為二進位檔或可執行檔。 二進位檔通常是編譯器與連結器的輸出結果，內含機器指令與程式資料。 `exec`（[kernel/exec.c:23](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/exec.c#L23)）會使用 `namei`（[kernel/exec.c:36](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/exec.c#L36)）開啟指定路徑 `path` 所對應的二進位檔，`namei` 的細節會在第八章中說明

接著它會讀取 ELF header，xv6 的二進位檔使用的是 ELF 格式，該格式定義於 [kernel/elf.h](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/elf.h#L6)。 一個 ELF 格式的檔案由一個 ELF header `struct elfhdr`，和一連串的 program section header `struct proghdr` 組成。 每個 `proghdr` 描述了應載入記憶體中的一個應用程式區段，xv6 的程式通常有兩個 program section header：一個用於指令段，一個用於資料段

第一個步驟是快速檢查該檔案是否可能是一個 ELF 的二進位檔。 一個 ELF 檔案的開頭會包含四個位元組的「魔術號」：`0x7F`、`'E'`、`'L'`、`'F'`，也可寫成 `ELF_MAGIC`。 如果 ELF header 的魔術號正確，`exec` 就會假設這個二進位檔格式正確無誤

`exec` 會透過 `proc_pagetable`（[kernel/exec.c:49](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/exec.c#L49)）建立一個不含任何使用者映射的新 page table，並透過 `uvmalloc`（[kernel/exec.c:65](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/exec.c#L65)）為每個 ELF segment 分配記憶體，再用 `loadseg` 將每個 segment 載入到記憶體中。 `loadseg`（[kernel/exec.c:10](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/exec.c#L10)）會使用 `walkaddr` 來找到已分配的記憶體的實體位址，以便寫入 ELF segment 內的每一個 page，並使用 `readi` 從檔案中讀取資料

`/init` 是第一個使用 `exec` 創建的使用者程式，其 program section header 如下：

```
# objdump -p user/_init

user/_init:     file format elf64-little

Program Header:
0x70000003 off    0x0000000000006bb0 vaddr 0x0000000000000000
                                       paddr 0x0000000000000000 align 2**0
         filesz 0x000000000000004a memsz 0x0000000000000000 flags r--
    LOAD off    0x0000000000001000 vaddr 0x0000000000000000
                                       paddr 0x0000000000000000 align 2**12
         filesz 0x0000000000001000 memsz 0x0000000000001000 flags r-x
    LOAD off    0x0000000000002000 vaddr 0x0000000000001000
                                       paddr 0x0000000000001000 align 2**12
         filesz 0x0000000000000010 memsz 0x0000000000000030 flags rw-
   STACK off    0x0000000000000000 vaddr 0x0000000000000000
                                       paddr 0x0000000000000000 align 2**4
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
```

我們可以看到，text 段應該從檔案中偏移量為 `0x1000` 的位置載入到記憶體中虛擬位址為 `0x0` 的位置，且不具有寫入權限。 我們也可以看到，data 段應該載入到 page 對齊的位址 `0x1000`，並且不具有執行權限

一個程式區段的 `filesz` 可能會小於 `memsz`，表示兩者之間的差距應該用零填滿（例如 C 的全域變數），而不是從檔案中讀取資料。 以 `/init` 為例，其資料段的 `filesz` 為 `0x10` bytes，而 `memsz` 為 `0x30` bytes，因此 `uvmalloc` 會配置足夠的實體記憶體以容納 `0x30` bytes，但僅會從 `/init` 檔案中讀取 `0x10` bytes 的資料

現在 `exec` 會分配並初始化 user stack，它只會分配一個 page 用作 stack。 `exec` 將每個字串引數依序複製到 stack 頂部，並將它們的指標記錄在 `ustack` 中。 它會在即將傳給 `main` 的 `argv` 清單末端放上一個 `null` 指標。 `argc` 與 `argv` 的值會透過系統呼叫的回傳路徑傳給 `main`：`argc` 會經由系統呼叫的回傳值傳遞，放在暫存器 `a0` 中； 而 `argv` 則透過該行程的 trapframe 中的 `a1` 欄位傳遞

`exec` 會在 stack page 的下方放置一個不可存取的 page，這樣若有程式試圖使用超過一個 page 的 stack 時就會產生錯誤。 這個不可存取的 page 也讓 `exec` 能夠處理引數過大的情況； 若發生這種情形，`exec` 所使用的 `copyout`（[kernel/vm.c:359](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L359)）函數會在把引數複製到 stack 時發現目標 page 無法存取，然後回傳 -1

在準備新的記憶體映像的過程中，若 `exec` 偵測到錯誤，例如無效的程式區段，它會跳轉到標籤 `bad`，釋放新建立的映像，並回傳 -1。 `exec` 必須等到確定這次系統呼叫會成功時，才會釋放舊有的映像：因為若舊映像已經被釋放，系統呼叫就無法再回傳 -1 給它。 `exec` 中所有的錯誤情況都只會發生在建立新映像的過程中，一旦映像建構完成（[kernel/exec.c:125](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/exec.c#L125)），`exec` 就可以正式切換到新的 page table 並釋放舊的 page table 了（[kernel/exec.c:129](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/exec.c#L129)）

`exec` 會依照 ELF 檔案所指定的位址，將其位元組資料載入到記憶體中。 由於使用者或行程可以在 ELF 檔中放入任意位址，因此 `exec` 存在風險，ELF 檔中的位址可能會指向 kernel 區域，無論是意外還是惡意行為。 對於不設防的 kernel 來說，其後果從系統崩潰到惡意破壞 kernel 隔離機制（即安全漏洞）都有可能發生

xv6 採取了一些檢查措施以避免這些風險。 例如，使用者可以製作一個 ELF 檔，讓 `ph.vaddr` 指向一個任意位址，並給 `ph.memsz` 一個足夠大的值，讓相加結果發生溢位並變成像 `0x1000` 這樣看似合理的值。 xv6 使用 `if(ph.vaddr + ph.memsz < ph.vaddr)` 來檢查這兩者相加時是否發生 64 位元整數溢位，以達到防禦的效果

在舊版的 xv6 中，使用者位址空間也包含了 kernel（雖然在 user mode 中無法讀寫），使用者可以選擇一個對應到 kernel 記憶體的位址，這樣 ELF 檔的資料就會被複製進 kernel。 這在 RISC-V 版本的 xv6 中不會發生，因為 kernel 擁有獨立的 page table； `loadseg` 會將資料載入行程的 page table，而非 kernel 的 page table 

對於 kernel 開發者來說，很容易會遺漏關鍵地檢查。 實際上在 kernel 發展歷史中，常常會因為檢查不足而讓使用者程式得以利用漏洞獲得 kernel 權限。 xv6 很可能也沒有完全驗證由使用者層傳入 kernel 的資料，這可能會被惡意的使用者程式加以利用，來繞過 xv6 的隔離機制

## 3.9 Real world

如同大多數的作業系統，xv6 使用 paging hardware 來進行記憶體保護與映射。 大多數作業系統會比 xv6 更複雜地使用 paging 技術，透過 paging 與 page 錯誤例外的結合來達成，這部分我們會在第四章內討論

xv6 簡化了實作，因為 kernel 直接使用虛擬位址與實體位址的一對一映射，並假設物理 RAM 位於 `0x80000000` 這個位址，同時也是 kernel 預期載入的位置。 這在使用 QEMU 時可以正常運作，但在真實硬體上卻是個壞主意，因為真實硬體的 RAM 與裝置會被配置在不可預測的物理位址上，例如在某些系統中，`0x80000000` 處可能根本沒有 RAM，而這正是 xv6 預期存放 kernel 的位置。 更嚴謹的 kernel 設計會利用 page table 將任意的物理記憶體配置映射成可預測的 kernel 虛擬位址佈局

RISC-V 支援針對實體位址層級的保護功能，但 xv6 並未使用這項功能

在擁有大量記憶體的機器上，使用 RISC-V 所支援的「super pages」是合理的。 但當實體記憶體很小時，使用小 page 比較合理，這樣可以以更細緻的粒度進行配置與 page-out 到硬碟。 例如，如果一個程式只用到 8 KB 記憶體，卻給它一整個 4 MB 的超大 page，那就很浪費。 在具備大量 RAM 的機器上，使用大 page 比較合理，並且可以減少管理 page table 的負擔

xv6 kernel 缺乏類似 `malloc` 的配置器來提供小型物件的記憶體空間，這使得 kernel 無法使用需要動態分配的複雜資料結構。 更精緻的 kernel 會配置多種大小的小型區塊，而不只是像 xv6 一樣僅使用 4096 位元組的區塊； 一個真正的 kernel 配置器需要同時處理大與小的記憶體分配需求

記憶體分配一直是個經久不衰的熱門議題，其基本問題是如何有效利用有限的記憶體，並為未來不可預期的請求做準備。 而如今人們更在意分配速度，而非空間效率

## 3.10 Exercises

1. 解析 RISC-V 的 device tree，以找出該電腦的實體記憶體容量
2. 撰寫一個使用者程式，透過呼叫 `sbrk(1)` 以增加一個位元組的位址空間。 執行程式，並觀察呼叫 `sbrk` 前後該程式的 page table。 kernel 實際上分配了多少空間？ 新分配的記憶體對應的 PTE 中包含了哪些內容？
3. 修改 xv6，使 kernel 可以使用 super pages
4. 傳統的 Unix 系統在 `exec` 實作中會對 shell script 加以特殊處理。 如果目標檔案的開頭是 `#!`，那麼首行會被視為用來解讀該檔案的程式。 例如，若 `exec` 被呼叫來執行 `myprog arg1`，而 `myprog` 的第一行是 `#!/interp`，那麼 `exec` 實際上會執行 `/interp myprog arg1`。 在 xv6 中實作這個行為
5. 為 kernel 實作地址空間佈局隨機化（ASLR）

## Bibliography

- <a id="1">[1]</a>：The RISC-V instruction set manual Volume II: privileged specification. https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link, 2024
