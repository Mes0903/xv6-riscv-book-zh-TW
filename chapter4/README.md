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

## 4.6 Page-fault exceptions

xv6 對於例外狀況的反應相當無趣：如果例外發生在 user space，kernel 就會把出錯的 process 給 kill 掉； 如果例外發生在 kernel，kernel 則會直接 panic。 真正的作業系統通常會用更有趣的方式來處理這些狀況

舉個例子，許多 kernel 會利用 page fault 來實作 copy-on-write（COW）型的 `fork`。 為了說明這種 `fork`，讓我們回到 xv6 的 `fork`，它在第三章內有被提到。 `fork` 會讓 child process 初始的記憶體內容和 parent process 當下的記憶體內容相同。 xv6 用 `uvmcopy`[（kernel/vm.c:313）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/vm.c#L313)來實作這個功能，它會為 child 配置實體記憶體，並把 parent 的記憶體內容複製過去。 如果能讓 parent 和 child 共享 parent 的實體記憶體，效率會更高。 不過直接這樣做是行不通的，因為他們會互相寫入共享的 stack 和 heap，導致彼此的執行出錯

只要搭配正確的 page table 權限設定與 page fault 機制，parent 和 child 是可以安全地共享實體記憶體的。 當某個虛擬位址沒有對應的 page table 映射，或該映射的 `PTE_V` 位元沒被設起來，或權限位元（`PTE_R`、`PTE_W`、`PTE_X`、`PTE_U`）不允許所嘗試的操作時，CPU 就會產生 page-fault exception。 在 RISC-V 架構中，page fault 分為三種類型：load page fault（由 `load` 指令引起）、store page fault（由 `store` 指令引起）和 instruction page fault（由 instruction fetch 造成）。 `scause` 暫存器會指出是哪種 page fault，而 `stval` 則會記錄無法被轉換的位址

COW `fork` 的基本做法是，讓 parent 和 child 一開始共享所有的 page frame，但他們各自都會將這些 page 設成唯讀的（`PTE_W` 欄位清 0）。 parent 和 child 都可以讀取這些共享記憶體，但如果任一方對某個 page 做了寫入操作，RISC-V CPU 就會產生一個 page-fault exception。 kernel 的 trap handler 會處理這個例外：它會配置一張新的 page frame，並把發生 fault 時那個位址對應的 page frame 的內容複製過去。 然後 kernel 會更新發生 fault 的 process 的 page table，把對應的 PTE 改成指向新的 page frame，並允許讀寫

最後 kernel 會讓該 process 從造成 fault 的那條指令重新執行。 而因為這時 PTE 已經允許寫入了，所以這次執行就不會再觸發 page fault。 copy-on-write 需要維護額外的資訊來追蹤哪些 page frame 可以被釋放，因為每張 page 可能會被多張 page table 所參考，這些參考會隨著 fork、page fault、exec 和 exit 而改變。 這樣的記錄機制還帶來一個重要的最佳化：如果某個 process 發生 store page fault，但該 page frame 只有被它自己的 page table 參照，那其實就不需要做複製

copy-on-write 可以讓 `fork` 更快，因為在 fork 的當下不需要複製記憶體。 雖然之後在寫入時還是可能得做複製，但實際上大多數記憶體都不需要真的被複製。 常見的一個例子是 `fork` 後馬上做 `exec`：在 `fork` 之後可能只有少數的 page 被寫入，而 child 的 `exec` 又會釋放掉大部分從 parent 繼承來的記憶體。 copy-on-write `fork` 可以避免複製這些記憶體。 此外，COW `fork` 是透明的，其不需要對應用程式做任何修改，它們就能自動受益

page table 和 page fault 的組合，除了能實作 COW `fork` 以外，還能支援很多有趣的功能。 其中一個被廣泛使用的機制是 lazy allocation，它包含兩個步驟。 第一步，當應用程式呼叫 `sbrk` 向系統要求更多記憶體時，kernel 會紀錄其要增加的大小，但不會馬上分配實體記憶體，也不會為這段新的虛擬位址區間建立 PTE。 第二步，當某個新位址上發生 page fault 時，kernel 才會真正配置一張 page frame，並把它映射進 page table。 就像 COW `fork` 一樣，lazy allocation 對應用程式來說也是透明的

由於應用程式請求的記憶體通常會比實際需要的還多，因此 lazy allocation 在這種情況下就非常有效：對於那些應用程式從未實際使用的 page，kernel 完全不需要做任何處理。 此外，如果應用程式一次請求大量的位址空間，而沒有 lazy allocation 的話，`sbrk` 的成本會非常高：例如應用程式要求 1GB 的記憶體時，kernel 必須配置並清零 262,144 個 4096-byte 的 page。 而 lazy allocation 能使這筆成本隨時間攤平

然而，lazy allocation 也會帶來額外的 page fault 開銷，因為每次 page fault 都會牽涉一次 user/kernel 的切換。 為了降低這項成本，作業系統可以在每次 page fault 時一次配置多個連續的 page，而不是只配置一個，並且還可以為這種 page fault 特化 kernel 的進出路徑

另一個廣泛使用且仰賴 page fault 的功能是 demand paging。 在 xv6 中，當執行 `exec` 時，它會在啟動應用程式前就把應用程式的 text 和 data 段全部載入記憶體。 由於應用程式可能很大，而從硬碟讀資料又很耗時，這個啟動成本對使用者而言可能會很明顯

為了縮短啟動時間，現代的 kernel 一開始並不會把可執行檔載入記憶體，而是建立一份 user page table，並將其中所有 PTE 標成 invalid。 kernel 啟動程式後，每當程式第一次使用某個 page，就會發生 page fault，然後 kernel 根據這個 fault 去從硬碟讀入該 page 的內容，並將它映射到 user 的位址空間。 就像 COW fork 和 lazy allocation 一樣，這個機制對應用程式來說是透明的

電腦上執行的程式還可能會需要超過實體記憶體容量的記憶體。 為了優雅地處理這種情況，作業系統可能會實作 swapping 的機制。 它的基本想法是：只在記憶體中保留部分使用者的 page，剩下的則儲存在硬碟中的 swap space。 kernel 會把那些對應到硬碟中的 swap space 的記憶體，其對應的 PTE 標為 invalid

接著如果應用程式試圖使用某個已經被 swap out 到硬碟的 page，便會觸發 page fault，此時該 page 必須被 swap in：kernel 的 trap handler 會配置一張實體記憶體，將對應的資料從硬碟讀回 RAM，然後更新對應的 PTE，讓它指向這張新的 page frame

如果某個 page 需要被 swap in，但當下已經沒有任何可用的實體記憶體了，這種情況下 kernel 必須先釋放出一張 page frame，方法是將其中的一個 page 做 swap out，也就是把它「搬移」到硬碟上的 swap space，並將所有參考該 page 的 PTE 標記為 invalid

不過「搬移」的成本很高，因此在它不常發生的情況下 paging 的表現較好，這代表應用程式只會使用其分配記憶體中的一小部分，而且這些常用 page 的總能被放在記憶體裡。 這種特性通常被稱為良好的 locality of reference。 就像其他許多虛擬記憶體技術一樣，kernel 通常會讓 swapping 對應用程式來說是透明的

::: tip  
這邊將原文的用詞改成了更常見的用詞：

- swapping：原文為 paging to disk
- swap out：原文為 paged out
- swap in：原文為 paged in

主要是因為對於「page」相關的詞我已經選擇保留原文了，這幾個再加進來會有些雜亂，導致不好閱讀  
:::

即使硬體提供了大量的記憶體，電腦在實際運作時仍經常處於幾乎沒有「空閒（free）」實體記憶體的狀態。 例如，雲端服務提供者通常會在單一機器上同時運行許多客戶的應用程式，以達到硬體資源的最大利用率。 再例如，使用者會在只有少量實體記憶體的智慧型手機上同時執行多個應用程式。 在這些情況下，每次配置一張新 page 前都可能需要先將某張現有的 page swap out。 因此，當實體記憶體資源緊張時，配置記憶體的成本會較高

在可用記憶體緊張、而程式實際上只使用其配置記憶體的一部分時，lazy allocation 和 demand paging 特別具有優勢。 這些技術還能避免某些情況下的資源浪費，例如：某個 page 被配置或從硬碟載入，但卻從未被實際使用，或甚至在使用前就被 swap out 了

還有一些其他功能同樣結合了 paging 和 page fault exception，例如自動延展的 stack 以及 memory-mapped file。 memory-mapped file 是指程式透過 `mmap` 系統呼叫把檔案映射進自己的位址空間，這樣程式就可以直接用 `load` 和 `store` 指令來讀寫這些檔案了

## 4.7 Real world

trampoline 與 trapframe 的設計看起來可能過於複雜。 背後的主要原因是 RISC-V 在觸發 trap 時會刻意地不做太多事，這樣可以讓 trap handler 的執行速度更快，而這點在實作上是非常重要的。 結果就是 kernel 的 trap handler 的前幾條指令必須在 user environment 下執行：使用的是 user page table，還有 user 的暫存器內容。 而且 trap handler 起初也不知道像「目前執行的 process 是誰」或「kernel page table 的位址」這些有用的資訊

這些問題之所以有解，是因為 RISC-V 提供了一些受保護的區域讓 kernel 可以在進入 user space 前先儲存資訊，例如 `sscratch` 暫存器，還有一些指向 kernel memory 的 user page table entry，但這些 entry 並沒有設 `PTE_U` 權限來保護。 xv6 的 trampoline 與 trapframe 就是善用了這些 RISC-V 的特性

如果 kernel memory 會被映射到每個 process 的 user page table 中（但不設 `PTE_U` 權限），那就不需要額外的 trampoline page 了。 這樣一來，從 user space trap 進 kernel 時也就不需要切換 page table。 這又讓 kernel 在實作 system call 時可以直接存取 user memory，因為這些記憶體已經被映射到了目前的 page table 中。 許多作業系統都會這樣設計來提升效率。 不過 xv6 為了避免 kernel 不小心使用 user pointer 而產生安全漏洞，也為了簡化 user 與 kernel 的位址空間不重疊所需要的處理，因此選擇不使用這種設計

真正的作業系統會實作像是 copy-on-write `fork`、lazy allocation、demand paging、paging to disk、memory-mapped file 等等機制。 此外，這些系統也會盡量讓整個實體記憶體都有用處，通常會拿來快取那些不屬於任何 process 的檔案內容

真正的作業系統也會提供一些 system call 讓應用程式來管理自己的位址空間，或是讓它自己處理 page fault，例如 `mmap`、`munmap`、`sigaction` 這些呼叫，也會提供像 `mlock` 這樣的呼叫來讓應用程式使用的 page 固定在記憶體裡而不被 swap out，或是像 `madvise` 這樣的呼叫讓應用程式告訴 kernel 它打算怎麼使用這塊記憶體

## 4.8 Exercises

1. `copyin` 和 `copyinstr` 會透過軟體的方式走訪 user page table。 設定 kernel 的 page table，讓 kernel 能直接映射 user program，這樣 `copyin` 和 `copyinstr` 就可以改用 `memcpy` 把 system call 的引數複製到 kernel space，而不用自己做 page table walk 了
2. 實作 lazy memory allocation
3. 實作 COW fork
4. 有沒有方法可以去掉每個 user address space 中的 `TRAPFRAME` page 映射？ 例如，`uservec` 是否可以改成直接把 32 個 user 暫存器 push 到 kernel stack，或是存在 `proc` 結構中？
5. 能不能改寫 xv6，讓它不需要 `TRAMPOLINE` page 的映射？
6. 實作 `mmap`

## Bibliography

- <a id="1">[1]</a>：The RISC-V instruction set manual Volume II: privileged specification. https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link, 2024
