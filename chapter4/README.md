---
title: xv6 riscv book chapter 4：Traps and system calls
date: 2025-07-31
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---

# xv6 riscv book chapter 4：Traps and system calls

有三種類型的事件會使 CPU 暫停正常的指令執行流程，並強制轉移控制權到一段專門處理該事件的程式碼。 第一種情況是系統呼叫，當使用者程式執行 `ecall` 指令時，會向 kernel 提出對應的要求。 第二種情況是例外（exception）：某條指令（無論是來自使用者或 kernel ）執行了非法操作，例如除以零或使用無效的虛擬位址。 第三種情況是裝置中斷（interrupt），如某個裝置發出訊號表示它需要被處理，例如硬碟完成某次讀寫請求的時候

本書將上述這些情況統稱為「trap」。 通常，發生 trap 時正在執行的程式碼之後需要能夠繼續執行，並且不應該察覺到任何特殊的事情發生了。 也就是說，我們通常希望 trap 是透明的； 這一點在處理裝置中斷時尤其重要，因為被中斷的程式碼通常不會預期到被打斷。 一般的處理流程是：trap 發生後控制權會轉移到 kernel； kernel 會儲存暫存器與其他狀態，以便之後能夠恢復執行； 接著 kernel 會執行對應的處理程式（例如系統呼叫的實作或裝置驅動程式）； 然後 kernel 會還原先前儲存的狀態並從 trap 返回； 最後原本的程式碼會從中斷處繼續執行

xv6 在 kernel 中處理所有的 trap，trap 並不會交由使用者程式處理。 將 trap 交由 kernel 處理對於系統呼叫來說是理所當然的。 而將中斷交由 kernel 處理也是合理的，因為有隔離的需求，所以只有 kernel 能夠操作裝置，而且 kernel 也提供了一個便利的機制，能夠讓多個行程共享裝置。 對於例外來說交由 kernel 處理也合理，因為 xv6 對於所有來自 user space 的例外都會以終止該程式作為回應

xv6 的 trap 處理流程分為四個階段：第一階段是 RISC-V CPU 執行的硬體動作； 第二階段是一些組合語言指令，用來為 kernel 的 C 程式碼做準備； 第三階段是一個 C 函式，它決定該如何處理這個 trap； 第四階段則是執行對應的系統呼叫或裝置驅動服務常式

儘管這三種 trap 類型有不少共通性，理論上 kernel 可以用一條通用的路徑來處理所有 trap，但實務上將其區分為兩種情況會更方便：來自 user space 的 trap，與來自 kernel 空間的 trap。 負責處理 trap 的 kernel 程式碼（不論是組語或 C）通常被稱為「handler」； 而最先執行的那幾條 handler 指令通常以組合語言撰寫，有時會被稱為「vector」

::: tip  
這個「vector」也有一些別的名字，如「trap prologue」或「trap entry」等，不過這些應該是口語上的名稱，而不是一個正式的名詞。 但從另外兩個名字你應該可以理解 xv6 中的「vector」就是一個統一的入口，發生 trap 時會先進入「vector」，然後再根據 trap 的種類去呼叫對應的 handler

具體而言，在 xv6 中有兩個「vector」：`uservec` 與 `kernelvec`，它們都是用組語寫的函式，兩者都只會先將必要的資訊存起來，然後呼叫對應的 C 函式 `usertrap` 與 `kerneltrap`，但這兩個 trap handler 的設計不太一樣，後面會再提到  
:::

## 4.1 RISC-V trap machinery

每個 RISC-V CPU 都有一組控制暫存器，kernel 會寫入這些暫存器以告訴 CPU 該如何處理 trap，並且 kernel 也可以讀取這些暫存器來得知 trap 的相關資訊，RISC-V 的官方文件中有完整的說明<sup>[[1]](#1)</sup>。 riscv.h（[kernel/riscv.h:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L1)）中包含了 xv6 使用的相關定義。 以下是幾個最重要的暫存器簡介：

- `stvec`：  
  - kernel 會在這裡寫入 trap handler 的位址； 當發生 trap 時，RISC-V 會跳到 `stvec` 所指定的位址執行處理該 trap 的 handler
- `sepc`：  
  - 當發生 trap 時，RISC-V 會將當下的程式計數器（`pc`）儲存在此處（因為 `pc` 隨即會被 `stvec` 的值覆蓋）。 `sret`（從 trap 返回的指令）會將 `sepc` 的內容複製回 `pc`。 kernel 也可以透過寫入 `sepc` 來控制 `sret` 返回的位置
- `scause`：  
  - RISC-V 會在此處寫入一個數值，描述這次 trap 的原因
- `sscratch`：  
  - trap handler 程式碼會使用 `sscratch` 來協助避免使用者的暫存器尚未被儲存前就被覆寫
- `sstatus`：  
  - 此暫存器中的 SIE 位元控制裝置中斷是否啟用。 如果 kernel 清除此位元，RISC-V 將會延後處理裝置中斷直到 kernel 再次設置它。 SPP 位元表示這次 trap 是從 user mode 還是 supervisor mode 進入的，並決定 `sret` 返回的模式

上述的這些都是 supervisor mode 下與處理 trap 有關的暫存器，並且在 user mode 中無法讀寫這些暫存器。 multi-core 晶片上的每個 CPU 都各有一組這些暫存器，並且任一時刻下都可能有多個 CPU 同時在處理 trap

當需要強制進入 trap 時，RISC-V 硬體會對所有 trap 類型執行以下動作：

1. 如果這次 trap 來自裝置中斷，且 `sstatus` 的 SIE 位元為清除狀態，則不執行以下步驟
2. 清除 `sstatus` 中的 SIE 位元以關閉中斷
3. 將 `pc` 的值複製到 `sepc`
4. 將當前的模式（user 或 supervisor）儲存在 `sstatus` 的 SPP 位元中
5. 將 `scause` 設定為此次 trap 的原因
6. 將執行模式設為 supervisor
7. 將 `stvec` 的值複製到 pc
8. 從新的 `pc` 開始執行

請注意，CPU 不會在 trap 發生時自動切換到 kernel 的 page table，也不會切換到 kernel stack，除了 `pc` 外也不會儲存任何其他暫存器。 這些任務必須由 kernel 的軟體來執行，CPU 在處理 trap 時只做最少的工作，主要是為了讓軟體有更多彈性； 例如，有些作業系統會在特定情況下省略切換 page table，以提升 trap 的效能

值得思考的是我們能否省略上述步驟中的某些部分，以更快速的處理 trap。 雖然在某些情況下簡化流程是可行的，但上方大多數的步驟若被省略會造成危險。 例如，假設 CPU 沒有切換程式計數器，那麼來自 user space 的 trap 就可能在仍執行使用者指令的情況下進入 supervisor 模式。 這些使用者指令可能會破壞使用者與 kernel 之間的隔離，例如修改 `satp` 暫存器指向允許存取整個實體記憶體的 page table。 因此，CPU 切換到 kernel 所指定的指令位址（即 `stvec`）是非常重要的

## 4.2 Traps from user space

xv6 會根據 trap 發生時是在 kernel space 中還是 user space 中而採取不同的處理方式。 這段會講述從 user code 發出的 trap 的流程； 至於 kernel code 發出的 trap，則會在第 4.5 節中說明

當執行緒正在 user space 執行時，如果 user program 發出了系統呼叫（透過 `ecall` 指令）、做了不合法的操作，或有裝置中斷發生，就可能會發生 trap。 從 user space 發出的 trap，其高階的處理路徑為：先進入 `uservec`（[kernel/trampoline.S:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L22)），接著進入 `usertrap`（[kernel/trap.c:37](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L37)）； 在處理完要返回 user space 時，會先經過 `usertrapret`（[kernel/trap.c:90](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L90)），最後再透過 `userret`（[kernel/trampoline.S:101](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L101)）回到 user program

xv6 的 trap 處理機制在設計上有個主要限制：RISC-V 硬體在觸發 trap 時並不會自動切換 page table。 這表示 `stvec` 中指向的 trap handler 位址，必須在 user page table 中有一個有效的映射，因為 trap 發生時仍是使用 user 的 page table 來執行。 此外，xv6 的 trap handler 還需要切換到 kernel 的 page table； 而為了讓 trap handler 在切換後能繼續執行，kernel page table 也必須對 `stvec` 所指向的 handler 有一份映射

xv6 透過 trampoline page 來滿足這些需求。 trampoline page 包含了 `uservec`，也就是 `stvec` 所指向的 trap handler。 xv6 會在每個 process 的 page table 中，將 trampoline page 映射到 `TRAMPOLINE` 這個位址； 這個位址在虛擬位址空間的最頂端，因此會高於 program 自己所使用的記憶體範圍

同時，trampoline page 也會在 kernel 的 page table 中被映射到相同的 `TRAMPOLINE` 位址，詳情可參考圖 2.3 和圖 3.3。 因為 trampoline page 有被映射進 user page table，且因為其在 kernel 的 page table 中也有映射，所以在發生 trap 而切換到 supervisor mode 時，handler 切換 page table 後仍能從該處繼續執行

::: tip  
這個被稱為 trampoline 的共享 page 會被映射進所有 process 的 user page table 和 kernel page table，而且都是映射到同一個虛擬位址。 這樣在 trap 發生時（使用 user page table）能進入 trampoline，而在 handler 中切換到 kernel page table 之後也不會失效，保證了執行的連貫性

Remark：

- root kernel page table 是全域唯一的，整個系統中只有一張
  - 任何 hart 只要處於 supervisor mode、且在執行 kernel 代碼時，`satp` 就指到這張表
  - 使用 direct mapping（VA == PA）
- root user page table 則是每個 process 都自己有一張
  
:::

`uservec` 這段 trap handler 的程式碼寫在 trampoline.S（[kernel/trampoline.S:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L22)）中。 當 `uservec` 開始執行時，32 個暫存器都還保留著被中斷的 user code 的值。 這 32 個值需要被存到記憶體中，好讓 kernel 在返回 user space 之前可以將它們還原。 但要把東西存到記憶體中，勢必得有一個暫存器來存放記憶體的位址，然而此時卻沒有任何通用暫存器可以用，對此 RISC-V 提供了一個解法：`sscratch` 暫存器。 `uservec` 開頭的 `csrw` 指令會先把 `a0` 存進 `sscratch`，這樣 `uservec` 就可以暫時直接使用 `a0` 了

`uservec` 接下來要做的事，就是把 32 個使用者暫存器全部存起來。 kernel 為每個 process 都配置了一個 page 給 `trapframe` 結構體，用來保存這些暫存器的值（[kernel/proc.h:43](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.h#L43)）。 由於現在 `satp` 還是指向 user page table，`uservec` 存取記憶體時，trapframe 必須要有對應的 user space 映射。 xv6 會在每個 process 的 user page table 中，把 `trapframe` 映射到 `TRAPFRAME` 這個虛擬位址，而這個位址就在 `TRAMPOLINE` 的正下方。 此外，process 的 `p->trapframe` 指標也會指向該 trapframe，不過是指向其實體位址，這樣 kernel 就能透過 kernel page table 存取它

因此 `uservec` 會將 `TRAPFRAME` 的位址載入到 `a0` 中，並將所有 user 暫存器的值存到那個位置，其中也包含剛剛存入 `sscratch` 內的 user 的 `a0` 值。 `trapframe` 中會包含當前 process 的 kernel stack 位址、目前 CPU 的 hartid、`usertrap` 函式的位址，以及 kernel page table 的位址。 `uservec` 會從中讀出這些資訊，接著把 `satp` 切換成 kernel page table，然後跳到 `usertrap`

::: tip  
`TRAMPOLINE` 和 `TRAPFRAME` 是兩個已經被寫死的 macro（kernel/memlayout.h:[44](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h#L44),[59](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h#L59)），換句話說每個 process 中 `TRAMPOLINE` 和 `TRAPFRAME` 的虛擬位址都是相同的。 區別在於對於不同的 process，`TRAMPOLINE` 都會對應到同一個 page frame，但對於不同的 process，`TRAPFRAME` 則會對應到不同的 page frame  
:::

`usertrap` 的工作是判斷 trap 的原因、處理它，然後返回（[kernel/trap.c:37](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L37)）。 它一開始會修改 `stvec`，這樣之後 kernel 裡如果再次發生 trap，就會進入 `kernelvec` 而不是 `uservec`。 接著會儲存 `sepc` 暫存器（也就是使用者程式的 PC），因為 `usertrap` 有可能呼叫 `yield` 去切換到其他 process 的 kernel thread，而那個 process 在切換回 user space 時會改寫 `sepc`

如果該 trap 是系統呼叫，`usertrap` 會呼叫 `syscall` 處理它； 如果是裝置中斷，就呼叫 `devintr`； 其他情況就是例外，kernel 會把出錯的 process 給 kill 掉。 系統呼叫的情況下，還會將儲存的 PC 加上 4，因為 RISC-V 的 `ecall` trap 發生後，`sepc` 仍會指向 `ecall` 那行指令，但 user code 恢復執行時需要從下一行繼續執行。 處理完要離開時，`usertrap` 會檢查這個 process 是否已經被 kill 了，或如果這次是 timer 中斷的話，是否應該交出 CPU

要返回 user space 的第一步是呼叫 `usertrapret`（[kernel/trap.c:90](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L90)），這個函式會設定 RISC-V 的控制暫存器，為之後從 user space 發生的 trap 做準備：包含將 `stvec` 設為 `uservec`，以及準備好 `uservec` 會用到的 trapframe 欄位。 `usertrapret` 也會把 `sepc` 復原為先前儲存的使用者程式計數器。 最後，`usertrapret` 會呼叫 `userret`，`userret` 這段程式碼也位在 trampoline page 上，且因為 `userret` 的組語程式會切換 page table，所以其會同時映射在 user 和 kernel page table 中

`usertrapret` 呼叫 `userret` 時，會把 process 的 user page table 位址傳入 `a0`（[kernel/trampoline.S:101](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L101)），`userret` 會將 `satp` 設成這份 user page table，記得 user page table 中的 kernel 區段只有 trampoline page 和 `TRAPFRAME` 會被映射，其他 kernel 區段都不會被映射

而 trampoline page 在 user 和 kernel page table 中擁有相同的虛擬位址，且映射到相同的實體位址，因此即使在這之後切換了 `satp`，`userret` 也還能繼續執行，在這之後 `userret` 就只能存取暫存器內容與 trapframe 的內容，`userret` 會將 `TRAPFRAME` 位址載入到 `a0`，用它來還原先前存下的使用者暫存器，還原使用者的 `a0`，最後執行 `sret` 指令返回 user space

::: tip  
`usertrapret` 呼叫 `userret` 的這段程式碼為：

```c
// tell trampoline.S the user page table to switch to.
uint64 satp = MAKE_SATP(p->pagetable);

// jump to userret in trampoline.S at the top of memory, which 
// switches to the user page table, restores user registers,
// and switches to user mode with sret.
uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64))trampoline_userret)(satp);
```

其中 `TRAMPOLINE` 為固定的 macro，值為 `0x3FFFFFF000`，這是一個固定的虛擬位址。 而 `userret` 與 `trampoline` 為定義在 trampoline.S 中的標籤位址，可以透過 nm 或 objdump 來看到具體的值：

```
mes@MesDesktop:~/xv6-riscv$ nm kernel/kernel | grep -E "(trampoline|userret)"
0000000080006000 T _trampoline
0000000080006000 T trampoline
000000008000609c T userret
mes@MesDesktop:~/xv6-riscv$ objdump -t kernel/kernel | grep -E "(trampoline|userret)"
0000000080006000 g       .text  0000000000000000 trampoline
000000008000609c g       .text  0000000000000000 userret
0000000080006000 g       .text  0000000000000000 _trampoline
```

因此 `trampoline` 的值為 `0x80006000`，`userret` 的值為 `0x8000609c`，這兩者也都為虛擬位址。 但注意這裡準備把 `satp` 換掉了，用的是 user page table，因此在 `userret` 內是「無法使用 direct mapping」的，也因此無法直接使用 `0x8000609c` 這個位址，就算他就是實際上的實體位址，但 user page table 內搞不好根本就沒有 `0x8000609c` 的這段映射，或是它可能會映射到其他 page frame

而前面有提到 trampoline page 會同時存在於 kernel 和 user 的 page table 中，這是 kernel page table 中加的 PTE：

```c
// map the trampoline for trap entry/exit to
// the highest virtual address in the kernel.
kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
```

這是 user page table 中加的 PTE：

```c
// Create a user page table for a given process, with no user memory,
// but with trampoline and trapframe pages.
pagetable_t
proc_pagetable(struct proc *p)
{
  ...
  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }
  ...
}
```

可以看到兩者會把 `TRAMPOLINE` 這個虛擬位址映射到同一個實體位址（`trampoline`）。 所以這邊用了 `TRAMPOLINE` 來走 Sv39 的路線將虛擬位址轉實體位址。 透過 `(userret - trampoline)` 求出偏移量，再加上 `TRAMPOLINE`，就可以得到位於 trampoline page 內的 VM 了，其值為 `0x3FFFFFF000 + 0x9c = 0x3FFFFFF09C`，透過 Sv39 的轉換，可以得到其值就為 `trampoline = 0x8000609c`

這邊比較容易卡住的點是 trampoline page 在 kernel page table 中有另外一種路徑是可以走 direct mapping 的。 換句話說如果 `satp` 指向的是 kernel page table，則 `trampoline` 和 `userret` 的值就同時代表了虛擬位址與實體位址，因為這兩個位址處於 kernel RAM 區段（`0x80000000` 至 `0x88000000`，見圖 3.3），因此在 kernel page table 中使用的是 direct mapping。 然而由於在 `userret` 的上下文忠 `satp` 指向的是 user page table，不能使用 direct mapping，所以才要繞這麼大一圈去計算實體位址  
:::

## 4.3 Code: Calling system calls

第二章最後提到了 `initcode.S` 會呼叫 `exec` 這個系統呼叫[（user/initcode.S:11）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/initcode.S#L11)。 現在我們來看看，這個來自 user space 的呼叫，是怎麼一路傳遞到 kernel 中對應的 `exec` 實作的

`initcode.S` 會把傳給 `exec` 的引數放進 `a0` 和 `a1` 暫存器中，並將系統呼叫的編號放進 `a7`。 系統呼叫編號會對應到 `syscalls` 陣列中的條目，這個陣列是一張函式指標的列表 [（kernel/syscall.c:107）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L107)。 接著 `ecall` 指令會觸發 trap 進入 kernel，如前所述，這會依序執行 `uservec`、`usertrap` 與 `syscall`

`syscall`[（kernel/syscall.c:132）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L132)會從 `trapframe` 中儲存的 `a7` 讀出系統呼叫編號，並利用它去查找 `syscalls` 陣列。 第一次的系統呼叫中，`a7` 裡會放 `SYS_exec`[（kernel/syscall.h:8）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.h#L8)，因此會呼叫到對應的實作函式 `sys_exec`

當 `sys_exec` 執行完畢並返回時，`syscall` 會把它的回傳值寫進 `p->trapframe->a0`。 這樣做的原因是，在 RISC-V 的 C 呼叫慣例中，回傳值會放在 `a0`，因此這樣可以讓原本 user space 中的 `exec()` 呼叫收到正確的回傳值。 慣例上，系統呼叫若發生錯誤會回傳負數，成功則是 0 或正數。 若系統呼叫編號不合法，`syscall` 會印出錯誤訊息，並回傳 -1

## 4.4 Code: System call arguments

kernel 中的系統呼叫實作需要取得 user code 所傳入的引數，由於 user code 會透過包裝函式來使用系統呼叫，這些引數一開始會依據 RISC-V 的 C 呼叫慣例放在暫存器中。 kernel 的 trap 處理流程會將使用者暫存器的內容儲存到目前 process 的 trapframe 中，讓 kernel code 之後可以從那裡找到它。 kernel 提供了 `argint`、`argaddr` 和 `argfd` 等函式，分別可用來從 trapframe 中讀取第 `n` 個系統呼叫的引數，並將其作為整數、指標或檔案描述符使用。 這些函式內部都會呼叫 `argraw`，以取得對應的使用者暫存器[（kernel/syscall.c:34）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L34)

有些系統呼叫會將指標作為引數傳入，而 kernel 必須使用這些指標去讀寫 user memory。 舉例來說，`exec`系統呼叫會傳給 kernel 一個陣列，裡面是指向 user space 中字串引數的指標。 這些指標會帶來兩個挑戰：第一是 user program 中可能有 bug 或是惡意程式碼，也可能會傳入一個無效的指標，甚至嘗試誘導 kernel 存取 kernel memory 而不是 user memory； 第二是 xv6 的 kernel page table 映射方式與 user page table 不同，因此 kernel 不能直接用一般的指令去從 user 的地址讀寫資料

kernel 提供了幾個函式，以安全地從 user 給的記憶體位址讀寫資料。 例如 `fetchstr`[（kernel/syscall.c:25）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/syscall.c#L25)，像 `exec` 這樣與檔案相關的系統呼叫會用 `fetchstr` 從 user space 讀取字串形式的檔名引數。 `fetchstr` 本身會呼叫 `copyinstr` 來完成底層的複製工作

`copyinstr`[（kernel/vm.c:415)）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L415)會從 user page table `pagetable` 中的虛擬位址 `srcva` 複製最多 `max` 位元組的資料到 `dst`。 由於 `pagetable` 並不是目前使用中的 page table，`copyinstr` 會使用 `walkaddr`（它會呼叫 `walk`）去查詢 `srcva` 在 `pagetable` 中對應的實體位址 `pa0`

由於 xv6 的 kernel page table 採用直接映射，這讓 `copyinstr` 可以直接從 `pa0` 複製字串到 `dst`。 `walkaddr`[（kernel/vm.c:109）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L109)也會檢查使用者給的虛擬位址是否真的屬於該 process 的 user 位址空間，這樣 kernel 就不會被誘騙去讀其他記憶體。 另一個類似的函式是 `copyout`，它會把資料從 kernel 複製到 user 提供的位址

## 4.5 Traps from kernel space

xv6 處理來自 kernel code 的 trap 的方式與處理 user code 的 trap 不同。 當進入 kernel 時，`usertrap` 會將 `stvec` 設定為指向 `kernelvec` 的組語程式碼[（kernel/kernelvec.S:12）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kernelvec.S#L12)。 由於 `kernelvec` 只有在 xv6 身處 kernel 狀態時才會被執行，所以 `kernelvec` 可以假設 `satp` 已經指向了 kernel page table，並且 stack pointer 也已經指向了一個合法的 kernel stack。 `kernelvec` 會把 32 個暫存器的值全部存入 stack 內，之後再從中還原，這樣就能讓被中斷的 kernel code 在不受干擾的情況下繼續執行

`kernelvec` 會將暫存器內容儲存在被中斷的 kernel thread 的 stack 上，因為這些暫存器的值本來就屬於該 thread。 這一點在 trap 導致切換到其他 thread 時特別重要，那種情況下 trap 結束後會從新 thread 的 stack 返回，而原本被中斷的 thread 的暫存器內容就安全地保留在它自己的 stack 上

在將暫存器存入 stack 之後，`kernelvec` 會跳到 `kerneltrap`[（kernel/trap.c:135）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L135)。 `kerneltrap` 主要用來處理兩種 trap：裝置中斷與例外狀況。 它會呼叫 `devintr`[（kernel/trap.c:185）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185)來偵測並處理裝置中斷。 如果這個 trap 不是裝置中斷，那就表示是例外，而在 xv6 的 kernel 中，只要發生例外就一律視為致命錯誤，此時 kernel 會呼叫 `panic` 並停止執行

如果這次的 `kerneltrap` 是由 `timer` 中斷觸發的，而且當前執行的是某個 process 的 kernel thread（不是 scheduler thread），那 `kerneltrap` 就會呼叫 `yield`，讓其他執行緒有機會被排程執行。 之後某個執行緒會再次呼叫 `yield`，使我們原本的執行緒與它的 `kerneltrap` 再度恢復執行。 `yield` 的詳細行為會在第七章說明

當 `kerneltrap` 處理完畢後，它需要返回到原本被 trap 中斷的那段程式碼。 由於 `yield` 可能已經修改了 `sepc` 和 `sstatus` 中的前一個模式，因此 `kerneltrap` 在開始時會先儲存這些暫存器，然後將它們還原並回到 `kernelvec`[（kernel/kernelvec.S:38）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/kernelvec.S#L38)中。 `kernelvec` 會從 stack 中將原先存入的暫存器取出，然後執行 `sret` 指令，這會把 `sepc` 的值寫回 `pc`，從而回到被中斷的 kernel code

這邊你可以想想看，如果 `kerneltrap` 是因為 timer 中斷而呼叫了 `yield`，那 trap 是怎麼完成返回的？

當某個 CPU 從 user space 進入 kernel 時，xv6 會把該 CPU 的 `stvec` 設定為 `kernelvec`； 你可以在 `usertrap` 中看到這段程式碼[（kernel/trap.c:29）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L29)。 不過在 kernel 開始執行且 `stvec` 還沒改成 `kernelvec` 的這段期間，`stvec` 仍指向 `uservec`，這段期間如果發生了裝置中斷就會有問題。 所幸 RISC-V 在進入 trap 時會自動關閉中斷，而 `usertrap` 也會等到設完 `stvec` 才重新開啟中斷

## Bibliography

- <a id="1">[1]</a>：The RISC-V instruction set manual Volume II: privileged specification. https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link, 2024
