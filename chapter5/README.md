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

需要作業系統處理的裝置通常可以將其設定成會產生 interrupt，kernel 的 trap handler 程式碼會辨認出裝置發出的 interrupt，並呼叫對應 driver 的 interrupt handler； 在 xv6 裡由 `devintr`（[kernel/trap.c:185](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185)） 來去呼叫對應的 driver

許多 device driver 的程式會在兩種情境下執行：一種是稱為 top half 的部分，會在 process 的 kernel thread 裡執行； 另一種是稱為 bottom half 的部分，會在 interrupt 發生時執行。 top half 會被像是 read 和 write 這類要讓裝置進行 I/O 的 system call 呼叫。 top half 的程式可能會要求硬體開始某個操作（例如請硬碟讀一個區塊），然後等待操作完成，接著會產生一個 interrupt。 driver 的 interrupt handler，也就是 bottom half，會判斷是哪個操作完成了，並在必要時喚醒等待的 process，然後告訴硬體可以開始執行下一個等待中的操作了

## 5.1 Code: Console input

console driver（[kernel/console.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c)）是一個簡單展示 driver 結構的範例，其透過接在 RISC-V 上的 UART 序列埠硬體來接收人類輸入的字元。 console driver 會一次累積一整行輸入，並處理像 backspace 和 control-u 這類的特殊字元。 像 shell 這樣的 user process，會透過 `read` system call 從 console 讀取一整行的輸入。 當你在 QEMU 中對 xv6 輸入時，你的按鍵會經由 QEMU 模擬的 UART 硬體傳送給 xv6

這個 driver 所操作的 UART 硬體，是由 QEMU 模擬出來的 16550 晶片<sup>[[1]](#1)</sup>。 在真實的電腦上，16550 晶片通常用來控制 RS232 序列連線，連接到終端機或另一台電腦。 而在執行 QEMU 時，它則連接到你的鍵盤與顯示器

以軟體的角度來看，UART 硬體是由 memory-mapped 的控制暫存器組成的。 也就是說，RISC-V 硬體會將某些實體位址對應到 UART 裝置，使得對那些位址的 load 與 store 操作實際上是與硬體互動，而不會存取 RAM。 UART 的 memory-mapped 位址從 `0x10000000` 開始，也就是 `UART0`（[kernel/memlayout.h:21](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/memlayout.h#L21)）

UART 有一些控制暫存器，每個暫存器的寬度都是一個 byte，它們相對於 `UART0` 的位移定義在（[kernel/uart.c:22](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L22)）裡面。 例如 `LSR` 暫存器裡的某些 bit 表示是否有輸入字元等著被軟體讀取，這些字元（如果有的話）可以從 RHR 暫存器讀出。 每次讀取後，UART 硬體會從它內部的 FIFO 中刪除該字元，當 FIFO 清空後，其會一併清除 `LSR` 中的 ready bit。 UART 的傳送邏輯與接收邏輯幾乎是獨立的，如果軟體寫入一個 byte 到 `THR`，UART 就會傳送該 byte

xv6 的 `main` 會呼叫 `consoleinit`（[kernel/console.c:182](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L182)）來初始化 UART 硬體。 這段程式會設定 UART，讓它在接收到每個輸入 byte 時產生接收中斷（receive interrupt），以及在每個輸出 byte 傳送完成時產生傳送完成中斷（transmit complete interrupt）（[kernel/uart.c:53](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L53)）

xv6 的 shell 會透過一個由 `init.c` 所開啟的 file descriptor（[user/init.c:19](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/init.c#L19)）來從 console 讀取資料。 對 `read` system call 的呼叫會一路進入 kernel，最後到達 `consoleread`（[kernel/console.c:80](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L80)）。 `consoleread` 會等待輸入透過中斷抵達，並將輸入暫存到 `cons.buf` 中，然後將輸入複製到 user space，並在整行輸入完成後才回傳給 user process。 若使用者尚未輸入完整的一行，任何呼叫 `read` 的 process 都會停在 `sleep`（[kernel/console.c:96](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L96)）呼叫中，第七章節會再詳細說明 `sleep` 的運作

當使用者輸入一個字元時，UART 硬體會要求 RISC-V CPU 產生一個中斷，這會啟動 xv6 的 trap handler。 trap handler 接著會呼叫 `devintr`（[kernel/trap.c:185](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185)），它會查看 RISC-V 的 `scause` 暫存器，以判斷該中斷是否來自外部裝置。 接著，它會向名為 PLIC 的硬體單元查詢是由哪個裝置產生的中斷（[kernel/trap.c:193](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L193)）； 對於 UART 產生的中斷，`devintr` 會呼叫 `uartintr`

`uartintr`（[kernel/uart.c:177](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L177)）會從 UART 硬體中讀取所有尚未處理的輸入字元，並將這些字元交給 `consoleintr`（[kernel/console.c:136](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L177)）； 它本身不會等待更多輸入，因為未來有新輸入時還會再次觸發新的中斷。 `consoleintr` 的工作是將這些輸入字元累積到 `cons.buf` 中，直到整行輸入完成為止。 `consoleintr` 會特別處理 backspace 與其他一些特殊字元

當輸入遇到 newline 時，`consoleintr` 會喚醒等待中的 `consoleread`（如果有的話）。 一旦被喚醒，`consoleread` 就會發現 `cons.buf` 中已經有一整行輸入，接著它會把這行資料複製到 user space，然後透過 system call 的機制將控制權返回給 user space

## Bibliography

- <a id="1">[1]</a>：Martin Michael and Daniel Durich. The NS16550A: UART design and application considerations. http://bitsavers.trailing-edge.com/components/national/_appNotes/AN-0491.pdf, 1987.
