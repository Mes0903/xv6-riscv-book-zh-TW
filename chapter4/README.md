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

有三種類型的事件會使 CPU 暫停正常的指令執行流程，並強制轉移控制權到一段專門處理該事件的程式碼。 第一種情況是系統呼叫，當使用者程式執行 `ecall` 指令時，會向核心提出對應的要求。 第二種情況是例外（exception）：某條指令（無論是來自使用者或核心）執行了非法操作，例如除以零或使用無效的虛擬位址。 第三種情況是裝置中斷（interrupt），如某個裝置發出訊號表示它需要被處理，例如硬碟完成某次讀寫請求的時候

本書將上述這些情況統稱為「trap」。 通常，發生 trap 時正在執行的程式碼之後需要能夠繼續執行，並且不應該察覺到任何特殊的事情發生了。 也就是說，我們通常希望 trap 是透明的； 這一點在處理裝置中斷時尤其重要，因為被中斷的程式碼通常不會預期到被打斷。 一般的處理流程是：trap 發生後控制權會轉移到核心； 核心會儲存暫存器與其他狀態，以便之後能夠恢復執行； 接著核心會執行對應的處理程式（例如系統呼叫的實作或裝置驅動程式）； 然後核心會還原先前儲存的狀態並從 trap 返回； 最後原本的程式碼會從中斷處繼續執行

xv6 在核心中處理所有的 trap，trap 並不會交由使用者程式處理。 將 trap 交由核心處理對於系統呼叫來說是理所當然的。 而將中斷交由核心處理也是合理的，因為有隔離的需求，所以只有核心能夠操作裝置，而且核心也提供了一個便利的機制，能夠讓多個行程共享裝置。 對於例外來說交由核心處理也合理，因為 xv6 對於所有來自使用者空間的例外都會以終止該程式作為回應

xv6 的 trap 處理流程分為四個階段：第一階段是 RISC-V CPU 執行的硬體動作； 第二階段是一些組合語言指令，用來為核心的 C 程式碼做準備； 第三階段是一個 C 函式，它決定該如何處理這個 trap； 第四階段則是執行對應的系統呼叫或裝置驅動服務常式

儘管這三種 trap 類型有不少共通性，理論上核心可以用一條通用的路徑來處理所有 trap，但實務上將其區分為兩種情況會更方便：來自使用者空間的 trap，與來自核心空間的 trap。 負責處理 trap 的核心程式碼（不論是組語或 C）通常被稱為「handler」； 而最先執行的那幾條 handler 指令通常以組合語言撰寫，有時會被稱為「vector」

## 4.1 RISC-V trap machinery

每個 RISC-V CPU 都有一組控制暫存器，核心會寫入這些暫存器以告訴 CPU 該如何處理 trap，並且核心也可以讀取這些暫存器來得知 trap 的相關資訊，RISC-V 的官方文件中有完整的說明<sup>[[1]](#1)</sup>。 riscv.h（[kernel/riscv.h:1](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L1)）中包含了 xv6 使用的相關定義。 以下是幾個最重要的暫存器簡介：

- `stvec`：  
  - 核心會在這裡寫入 trap handler 的位址； 當發生 trap 時，RISC-V 會跳到 `stvec` 所指定的位址執行處理該 trap 的 handler
- `sepc`：  
  - 當發生 trap 時，RISC-V 會將當下的程式計數器（`pc`）儲存在此處（因為 `pc` 隨即會被 `stvec` 的值覆蓋）。 `sret`（從 trap 返回的指令）會將 `sepc` 的內容複製回 `pc`。 核心也可以透過寫入 `sepc` 來控制 `sret` 返回的位置
- `scause`：  
  - RISC-V 會在此處寫入一個數值，描述這次 trap 的原因
- `sscratch`：  
  - trap handler 程式碼會使用 `sscratch` 來協助避免使用者的暫存器尚未被儲存前就被覆寫
- `sstatus`：  
  - 此暫存器中的 SIE 位元控制裝置中斷是否啟用。 如果核心清除此位元，RISC-V 將會延後處理裝置中斷直到核心再次設置它。 SPP 位元表示這次 trap 是從 user mode 還是 supervisor mode 進入的，並決定 `sret` 返回的模式

上述的這些都是 supervisor mode 下與處理 trap 有關的暫存器，並且在 user mode 中無法讀寫這些暫存器。 這些暫存器，多核心晶片上的每個 CPU 自己都有獨立的一組，並且在任一時刻都可能有多個 CPU 同時在處理 trap

當需要強制進入 trap 時，RISC-V 硬體會對所有 trap 類型執行以下動作：

1. 如果這次 trap 來自裝置中斷，且 `sstatus` 的 SIE 位元為清除狀態，則不執行以下步驟
2. 清除 `sstatus` 中的 SIE 位元以關閉中斷
3. 將 `pc` 的值複製到 `sepc`
4. 將當前的模式（使用者或監督者）儲存在 `sstatus` 的 SPP 位元中
5. 將 `scause` 設定為此次 trap 的原因
6. 將執行模式設為 supervisor
7. 將 `stvec` 的值複製到 pc
8. 從新的 `pc` 開始執行

請注意，CPU 不會在 trap 發生時自動切換到核心的 page table，也不會切換到 kernel stack，除了 `pc` 外也不會儲存任何其他暫存器。 這些任務必須由核心的軟體來執行，CPU 在處理 trap 時只做最少的工作，主要是為了讓軟體有更多彈性； 例如，有些作業系統會在特定情況下省略切換 page table，以提升 trap 的效能

值得思考的是我們能否省略上述步驟中的某些部分，以更快速的處理 trap。 雖然在某些情況下簡化流程是可行的，但大多數步驟若被省略會造成危險。 例如，假設 CPU 沒有切換程式計數器，那麼來自 user space 的 trap 就可能在仍執行使用者指令的情況下進入 supervisor 模式。 這些使用者指令可能會破壞使用者與核心之間的隔離，例如修改 `satp` 暫存器指向允許存取整個實體記憶體的 page table。 因此，CPU 切換到核心所指定的指令位址（即 `stvec`）是非常重要的

## Bibliography

- <a id="1">[1]</a>：The RISC-V instruction set manual Volume II: privileged specification. https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link, 2024