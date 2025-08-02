---
title: xv6 riscv book chapter 7：Scheduling
date: 2025-08-03
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---

# xv6 riscv book chapter 7：Scheduling

任何作業系統在執行時，通常會有比電腦實際擁有的 CPU 數量還多的 process，因此必須有一套計劃來讓這些 process 能夠輪流使用 CPU。 理想上，這種共享機制對使用者的 process 來說應該要是透明的。 常見的做法是透過「multiplexing」的方式，把多個 process 映射（multiplex）到實際的硬體 CPU 上，讓每個 process 產生擁有自己虛擬 CPU 的錯覺。 本章會說明 xv6 是如何實現這樣的 multiplexing 的

## 7.1 Multiplexing

xv6 的 multiplexing 機制會在兩種情況下讓某個 CPU 從一個 process 切換到另一個。 第一種是當 process 呼叫會阻塞（也就是需要等待某些事件才能繼續）的 system call 時，例如 `read`、`wait` 或 `sleep`，此時會透過 xv6 的 `sleep` 與 `wakeup` 機制來切換； 第二種是為了應對那些長時間運算而不會阻塞的 process，xv6 會定期強制讓 CPU 切換到其他 process。 前者稱為「自願切換（voluntary switches）」，後者則稱為「非自願切換（involuntary switches）」。 透過這些切換，xv6 創造出每個 process 各自擁有一顆 CPU 的錯覺

要實作 multiplexing 有幾個挑戰：

- 第一，該如何從一個 process 切換到另一個？
  - 基本的作法是儲存與還原 CPU 的暫存器，不過因為這種行為無法用 C 來表達，所以會比較麻煩
- 第二，該如何讓「強制切換」對 user process 來說是透明的？
  - xv6 採用了一個標準技巧，由硬體 timer 所觸發的中斷來驅動 context switch
- 第三，由於所有 CPU 都會在同一組 process 間切換，因此必須設計一套鎖定策略以避免 race condition
- 第四，當一個 process 結束時，它的記憶體與其他資源必須被釋放，但 process 自己無法完成這些釋放動作
  - 例如 process 無法在其還在使用 kernel stack 的情況下釋放那塊 stack，需要其他 thread 幫它收尾
- 第五，對於多核機器，每顆 CPU 都必須記住自己目前正在執行哪個 process，這樣 system call 才能正確地作用在那個 process 的 kernel 狀態上
- 最後，`sleep` 和 `wakeup` 機制允許 process 放棄 CPU，並等待其他 process 或中斷將它喚醒，而這裡需要特別小心，避免 race condition 導致喚醒通知被遺失

## 7.2 Code: Context switching

圖 7.1 說明了從一個 user process 切換到另一個時所經歷的步驟：首先是從 user space 發出 trap（可能是 system call 或 interrupt），轉入舊 process 的 kernel thread； 接著切換到目前 CPU 的 scheduler thread； 然後切換到新 process 的 kernel thread； 最後從 trap return 回到新的 user-level process

xv6 為 scheduler 使用了獨立的 thread（各自擁有暫存器與 stack 的保存空間），因為讓 scheduler 在任意 process 的 kernel stack 上執行並不安全：其他 CPU 可能會在這段期間喚醒該 process 並開始執行，若兩個 CPU 共用同一個 stack，會造成災難性的後果。 為了處理多顆 CPU 同時執行、並有 process 要放棄 CPU 的情況，xv6 為每個 CPU 配置了獨立的 scheduler thread。 在這一節中，我們將會詳細探討 kernel thread 與 scheduler thread 之間切換的具體實作方式

![Figure 7.1: Switching from one user process to another. In this example, xv6 runs with one CPU (and thus one scheduler thread).](image/switch.png)

從一個 thread 切換到另一個 thread 的過程，需要將舊 thread 的 CPU 暫存器儲存下來，並還原新 thread 先前儲存的那些暫存器。 由於 stack pointer 和 program counter 都會被儲存與還原，這也表示 CPU 將會切換至新的 stack，並且執行新的程式碼

`swtch` 函式負責儲存與還原暫存器，實作 kernel thread 間的切換。 `swtch` 並不直接知道所謂的「thread」是什麼，它只負責儲存與還原一組 RISC-V 的暫存器，這組暫存器的集合被稱作「context」。 當某個 process 要放棄 CPU 時，它的 kernel thread 會呼叫 `swtch`，把自己的 context 儲存起來，並還原 scheduler 的 context

每個 context 都存在 `struct context`（[kernel/proc.h:2](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.h#L2)）裡，這個結構會被放在某個 process 的 `struct proc` 或某個 CPU 的 `struct cpu` 中。 `swtch` 會接收 `struct context *old` 和 `struct context *new` 這兩個引數，並將目前的暫存器儲存到 `old`，從 `new` 中載入先前儲存的暫存器，然後 return

現在讓我們來追蹤一個 process 如何透過 `swtch` 切換進入 `scheduler` 的。 在第四章中我們看到，中斷的結束階段中有一種情況是 `usertrap` 呼叫 `yield`。 而 `yield` 接著會呼叫 `sched`，`sched` 再呼叫 `swtch`，把目前的 context 存到 `p->context` 中，並切換到先前儲存在 `cpu->context` 的 scheduler context（[kernel/proc.c:506](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L506)）

`swtch`（[kernel/swtch.S:3](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/swtch.S#L3)）只會儲存 callee-saved 暫存器，而 caller-saved 暫存器則會由 C 編譯器在呼叫端負責儲存到 `stack` 上。 `swtch` 知道每個暫存器對應到 `struct context` 中的哪個成員，以及該成員的偏移量。 它不會儲存 program counter，而是儲存 `ra` 暫存器，這個暫存器中存放的是呼叫 `swtch` 那一行指令的 return address

接著，`swtch` 從新的 context 還原暫存器，這些值是先前某次 `swtch` 儲存的。 當 `swtch` 呼叫 `ret` 返回時，它會回到還原後的 `ra` 所指向的那行指令，也就是新 thread 先前呼叫 `swtch` 的那個位置。 同時，因為 `sp` 已被還原為新 thread 的 stack pointer，因此執行也會從新 thread 的 stack 上繼續

在我們這個例子中，`sched` 會呼叫 `swtch`，並切換到 `cpu->context`，也就是這顆 CPU 專屬的 scheduler context。 這份 context 是在先前某個時刻，由 scheduler 呼叫 `swtch` 並切換到現在這個 process 時所儲存的（[kernel/proc.c:466](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/proc.c#L466)）。 所以當我們目前追蹤的這次 `swtch` return 時，它實際上不是回到 `sched`，而是回到 scheduler，而且此時 stack pointer 已經是這顆 CPU 的 scheduler stack 了

::: tip  
根據 [RISC-V Calling Conventions](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/blob/master/riscv-cc.adoc)，暫存器口語上會分成兩類：

- caller-saved registers：呼叫者在呼叫 function 前要自己備份，包含 `a0–a7`, `t0–t6`, `ra` 等
- callee-saved registers：被呼叫者（像 `swtch`）必須保存與還原，包含 `s0–s11`, `sp` 等

這可以在點進去一開始的表格中的「Preserved across calls?」欄位看到，為「Yes」的就是文中說的 callee-saved register。 而 `ra` 雖然不是 callee-saved register，但 `swtch` 為了做 context switch 所以有存

而 `swtch` 做的事基本上就是：

1. 把被換出的 process 的 context 存到 `struct context *old`
2. 把換進來要執行的 process 的 context 用 `struct context *new` 的內容復原
3. 利用 `ret` 回到 `new->ra` 處執行

而對於文中的例子，由於它是由 `sched` 去呼叫 `swtch`，所以 `new` 填的會是 `&mycpu()->context`，也就是一個 pre-CPU 的 scheduler context  
:::
