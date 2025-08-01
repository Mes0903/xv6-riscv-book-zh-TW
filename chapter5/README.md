---
title: xv6 riscv book chapter 5：Interrupts and device drivers
date: 2025-08-01
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
---

# xv6 riscv book chapter 5：Interrupts and device drivers

driver 是作業系統中負責管理特定裝置的程式碼：它會設定裝置的硬體、命令裝置執行操作、處理裝置產生的中斷，並與那些可能正在等待該裝置 I/O 的 process 互動。 driver 的程式碼常常很棘手，因為它與其所管理的裝置是並行執行的。 此外，driver 還必須理解裝置的硬體介面，而這些介面可能很複雜，也可能沒有良好文件說明

需要作業系統處理的裝置通常可以將其設定成會產生 interrupt，kernel 的 trap handler 程式碼會辨認出裝置發出的 interrupt，並呼叫對應 driver 的 interrupt handler； 在 xv6 裡由 `devintr`[（kernel/trap.c:185）](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185) 來去呼叫對應的 driver

許多 device driver 的程式會在兩種情境下執行：一種是稱為 top half 的部分，會在 process 的 kernel thread 裡執行； 另一種是稱為 bottom half 的部分，會在 interrupt 發生時執行。 top half 會被像是 read 和 write 這類要讓裝置進行 I/O 的 system call 呼叫。 top half 的程式可能會要求硬體開始某個操作（例如請硬碟讀一個區塊），然後等待操作完成，接著會產生一個 interrupt。 driver 的 interrupt handler，也就是 bottom half，會判斷是哪個操作完成了，並在必要時喚醒等待的 process，然後告訴硬體可以開始執行下一個等待中的操作了
