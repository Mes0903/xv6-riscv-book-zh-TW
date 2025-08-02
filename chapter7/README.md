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
