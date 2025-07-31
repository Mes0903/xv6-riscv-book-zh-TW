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

本書將上述這些情況統稱為「trap」。 通常，發生 trap 時正在執行的程式碼之後需要能夠繼續執行，並且不應該察覺到任何特殊的事情發生了。 也就是說，我們通常希望 trap 是透明的； 這一點在處理裝置中斷時尤其重要，因為被中斷的程式碼通常不會預期到被打斷。 一般的處理流程是：trap 發生後控制權會轉移到 kernel ；  kernel 會儲存暫存器與其他狀態，以便之後能夠恢復執行； 接著 kernel 會執行對應的處理程式（例如系統呼叫的實作或裝置驅動程式）； 然後 kernel 會還原先前儲存的狀態並從 trap 返回； 最後原本的程式碼會從中斷處繼續執行

xv6 在 kernel 中處理所有的 trap，trap 並不會交由使用者程式處理。 將 trap 交由 kernel 處理對於系統呼叫來說是理所當然的。 而將中斷交由 kernel 處理也是合理的，因為有隔離的需求，所以只有 kernel 能夠操作裝置，而且 kernel 也提供了一個便利的機制，能夠讓多個行程共享裝置。 對於例外來說交由 kernel 處理也合理，因為 xv6 對於所有來自 user space 的例外都會以終止該程式作為回應

xv6 的 trap 處理流程分為四個階段：第一階段是 RISC-V CPU 執行的硬體動作； 第二階段是一些組合語言指令，用來為 kernel 的 C 程式碼做準備； 第三階段是一個 C 函式，它決定該如何處理這個 trap； 第四階段則是執行對應的系統呼叫或裝置驅動服務常式

儘管這三種 trap 類型有不少共通性，理論上 kernel 可以用一條通用的路徑來處理所有 trap，但實務上將其區分為兩種情況會更方便：來自 user space 的 trap，與來自 kernel 空間的 trap。 負責處理 trap 的 kernel 程式碼（不論是組語或 C）通常被稱為「handler」； 而最先執行的那幾條 handler 指令通常以組合語言撰寫，有時會被稱為「vector」

## 4.1 RISC-V trap machinery

每個 RISC-V CPU 都有一組控制暫存器， kernel 會寫入這些暫存器以告訴 CPU 該如何處理 trap，並且 kernel 也可以讀取這些暫存器來得知 trap 的相關資訊，RISC-V 的官方文件中有完整的說明<sup>[[1]](#1)</sup>。 riscv.h（[kernel/riscv.h:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L1)）中包含了 xv6 使用的相關定義。 以下是幾個最重要的暫存器簡介：

- `stvec`：  
  -  kernel 會在這裡寫入 trap handler 的位址； 當發生 trap 時，RISC-V 會跳到 `stvec` 所指定的位址執行處理該 trap 的 handler
- `sepc`：  
  - 當發生 trap 時，RISC-V 會將當下的程式計數器（`pc`）儲存在此處（因為 `pc` 隨即會被 `stvec` 的值覆蓋）。 `sret`（從 trap 返回的指令）會將 `sepc` 的內容複製回 `pc`。  kernel 也可以透過寫入 `sepc` 來控制 `sret` 返回的位置
- `scause`：  
  - RISC-V 會在此處寫入一個數值，描述這次 trap 的原因
- `sscratch`：  
  - trap handler 程式碼會使用 `sscratch` 來協助避免使用者的暫存器尚未被儲存前就被覆寫
- `sstatus`：  
  - 此暫存器中的 SIE 位元控制裝置中斷是否啟用。 如果 kernel 清除此位元，RISC-V 將會延後處理裝置中斷直到 kernel 再次設置它。 SPP 位元表示這次 trap 是從 user mode 還是 supervisor mode 進入的，並決定 `sret` 返回的模式

上述的這些都是 supervisor mode 下與處理 trap 有關的暫存器，並且在 user mode 中無法讀寫這些暫存器。 這些暫存器，多 kernel 晶片上的每個 CPU 自己都有獨立的一組，並且在任一時刻都可能有多個 CPU 同時在處理 trap

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

值得思考的是我們能否省略上述步驟中的某些部分，以更快速的處理 trap。 雖然在某些情況下簡化流程是可行的，但大多數步驟若被省略會造成危險。 例如，假設 CPU 沒有切換程式計數器，那麼來自 user space 的 trap 就可能在仍執行使用者指令的情況下進入 supervisor 模式。 這些使用者指令可能會破壞使用者與 kernel 之間的隔離，例如修改 `satp` 暫存器指向允許存取整個實體記憶體的 page table。 因此，CPU 切換到 kernel 所指定的指令位址（即 `stvec`）是非常重要的

## 4.2 Traps from user space

xv6 會根據 trap 發生時是在 kernel 中還是 user code 中執行而採取不同的處理方式。 這段會講述從 user code 發出的 trap 的流程； 至於 kernel code 發出的 trap，則會在第 4.5 節中說明

當執行緒正在 user space 執行時，如果 user program 發出系統呼叫（透過 `ecall` 指令）、做了不合法的操作，或有裝置中斷發生，就可能會發生 trap。 從 user space 發出的 trap，其高階的處理路徑為：先進入 `uservec`（ [kernel/trampoline.S:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L22)），接著進入 `usertrap`（[kernel/trap.c:37](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L37)）； 處理完後要返回 user space 時，會先經過 `usertrapret`（[kernel/trap.c:90](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L90)），最後再透過 `userret`（[kernel/trampoline.S:101](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L101)）回到 user program

xv6 的 trap 處理機制在設計上有個主要限制：RISC-V 硬體在觸發 trap 時並不會自動切換 page table。 這表示 `stvec` 中指向的 trap handler 位址，必須在 user page table 中有一個有效的映射，因為 trap 發生時仍是使用 user 的 page table 來執行。 此外，xv6 的 trap handler 還需要切換到 kernel 的 page table； 而為了讓 trap handler 在切換後能繼續執行，kernel page table 也必須對 `stvec` 所指向的 handler 有一份映射

xv6 透過 trampoline page 來滿足這些需求。 trampoline page 包含了 `uservec`，也就是 `stvec` 所指向的 trap handler。 xv6 會在每個 process 的 page table 中，將 trampoline page 映射到 `TRAMPOLINE` 這個位址； 這個位址在虛擬位址空間的最頂端，因此會高於 program 自己所使用的記憶體範圍

同時，trampoline page 也會在 kernel 的 page table 中被映射到相同的 `TRAMPOLINE` 位址，詳情可參考圖 2.3 和圖 3.3。 因為 trampoline page 有被映射進 user page table，所以發生 trap 時可以在 supervisor mode 下從這裡開始執行； 又因為其在 kernel 的 page table 中也有映射，所以 handler 切換 page table 後仍能繼續執行

`uservec` 這段 trap handler 的程式碼寫在 trampoline.S（[kernel/trampoline.S:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L22)）中。 當 `uservec` 開始執行時，32 個暫存器都還保留著被中斷的 user code 的值。 這 32 個值需要被存到記憶體中，好讓 kernel 在返回 user space 之前可以將它們還原。 但要把東西存到記憶體中，勢必得有一個暫存器來存放記憶體的位址，然而此時卻沒有任何通用暫存器可以用，對此 RISC-V 提供了一個解法：`sscratch` 暫存器。 `uservec` 開頭的 `csrw` 指令會先把 `a0` 存進 `sscratch`，這樣 `uservec` 就可以暫時直接使用 `a0` 了

`uservec` 接下來要做的事，就是把 32 個 user registers 全部存起來。 kernel 為每個 process 都配置了一個 page 給 `trapframe` 結構體，用來保存這些暫存器的值（[kernel/proc.h:43](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.h#L43)）。 由於現在 `satp` 還是指向 user page table，`uservec` 存取記憶體時，trapframe 必須要有對應的 user space 映射。 xv6 會在每個 process 的 user page table 中，把 `trapframe` 映射到 `TRAPFRAME` 這個虛擬位址，而這個位址就在 `TRAMPOLINE` 的正下方。 此外，process 的 `p->trapframe` 指標也會指向該 trapframe，不過是指向其實體位址，這樣 kernel 就能透過 kernel page table 存取它

因此 `uservec` 會將 `TRAPFRAME` 的位址載入到 `a0` 中，並將所有 user 暫存器的值存到那個位置，其中也包含從 `sscratch` 讀回的 user 的 `a0`。 `trapframe` 中會包含當前 process 的 kernel stack 位址、目前 CPU 的 hartid、`usertrap` 函式的位址，以及 kernel page table 的位址。 `uservec` 會從中讀出這些資訊，接著把 `satp` 切換成 kernel page table，然後跳到 `usertrap`

`usertrap` 的工作是判斷 trap 的原因、處理它，然後返回（[kernel/-trap.c:37](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L37)）。 它一開始會修改 `stvec`，這樣之後 kernel 裡如果再次發生 trap，就會進入 `kernelvec` 而不是 `uservec`。 接著會儲存 `sepc` 暫存器（也就是使用者程式的 PC），因為 `usertrap` 有可能呼叫 `yield` 去切換到其他 process 的 kernel thread，而那個 process 在切換回 user space 時會改寫 `sepc`

如果該 trap 是系統呼叫，`usertrap` 會呼叫 `syscall` 處理它； 如果是裝置中斷，就呼叫 `devintr`； 其他情況就是例外，kernel 會把出錯的 process 給 kill 掉。 系統呼叫的情況下，還會將儲存的 PC 加上 4，因為 RISC-V 的 `ecall` trap 發生後，`sepc` 仍會指向 `ecall` 那行指令，但 user code 恢復執行時需要從下一行繼續執行。 處理完要離開時，`usertrap` 會檢查這個 process 是否已經被 kill 了，或如果這次是 timer 中斷的話，是否應該交出 CPU

要返回 user space 的第一步是呼叫 `usertrapret`（[kernel/trap.c:90](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L90)），這個函式會設定 RISC-V 的控制暫存器，為之後從 user space 發生的 trap 做準備：包含將 `stvec` 設為 `uservec`，以及準備好 `uservec` 會用到的 trapframe 欄位。 `usertrapret` 也會把 `sepc` 復原為先前儲存的使用者程式計數器。 最後，`usertrapret` 會呼叫 `userret`，這段程式碼位在 trampoline page 上，且因為 `userret` 的組語程式會切換 page table，所以其會同時映射在 user 和 kernel page table 中

`usertrapret` 呼叫 `userret` 時，會把 process 的 user page table 位址傳入 `a0`（[kernel/trampoline.S:101](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trampoline.S#L101)），`userret` 會將 `satp` 設成這份 user page table，記得 user page table 中的 kernel 區段只有 trampoline page 和 `TRAPFRAME` 會被映射，其他 kernel 區段都不會被映射

而 trampoline page 在 user 和 kernel page table 中都映射到相同的虛擬位址，因此即使在這之後切換了 `satp`，`userret` 也還能繼續執行，在這之後 `userret` 就只能存取暫存器內容與 trapframe 的內容，`userret` 會將 `TRAPFRAME` 位址載入到 `a0`，用它來還原先前存下的使用者暫存器，還原使用者的 `a0`，最後執行 `sret` 指令返回 user space

## Bibliography

- <a id="1">[1]</a>：The RISC-V instruction set manual Volume II: privileged specification. https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link, 2024