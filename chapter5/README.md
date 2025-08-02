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

xv6 的 shell 會透過一個由 init.c 所開啟的 file descriptor（[user/init.c:19](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/init.c#L19)）來從 console 讀取資料。 對 `read` system call 的呼叫會一路進入 kernel，最後到達 `consoleread`（[kernel/console.c:80](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L80)）。 `consoleread` 會等待輸入透過中斷抵達，並將輸入暫存到 `cons.buf` 中，然後將輸入複製到 user space，並在整行輸入完成後才回傳給 user process。 若使用者尚未輸入完整的一行，任何呼叫 `read` 的 process 都會停在 `sleep`（[kernel/console.c:96](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/console.c#L96)）呼叫中，第七章節會再詳細說明 `sleep` 的運作

::: tip  
`read` 系統呼叫的實作為 `sys_read`，內部最後會呼叫 `fileread`，而 `fileread` 會根據 `file` 這個結構體內的成員 `major` 來判斷要怎麼讀取這個 file descriptor：

```c
// Read from file f.
// addr is a user virtual address.
int
fileread(struct file *f, uint64 addr, int n)
{
  ...
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(1, addr, n);
  ...
  return r;
}
```

而在 `consoleinit` 中會將它接到 `consoleread`：

```c
void
consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```

而前面有說 init.c 會開一個 file descriptor 來從 console 讀取資料，其底層就是對應到 `CONSOLE`。 因此可知讀取 console 的流程為：`read` → `fileread` → `consoleread`  
:::

當使用者輸入一個字元時，UART 硬體會要求 RISC-V CPU 產生一個中斷，這會啟動 xv6 的 trap handler。 trap handler 接著會呼叫 `devintr`（[kernel/trap.c:185](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L185)），它會查看 RISC-V 的 `scause` 暫存器，以判斷該中斷是否來自外部裝置。 接著，它會向名為 PLIC 的硬體單元查詢是由哪個裝置產生的中斷（[kernel/trap.c:193](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L193)）； 對於 UART 產生的中斷，`devintr` 會呼叫 `uartintr`

`uartintr`（[kernel/uart.c:177](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L177)）會從 UART 硬體中讀取所有尚未處理的輸入字元，並將這些字元交給 `consoleintr`（[kernel/console.c:136](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L177)）； 它本身不會等待更多輸入，因為未來有新輸入時還會再次觸發新的中斷。 `consoleintr` 的工作是將這些輸入字元累積到 `cons.buf` 中，直到整行輸入完成為止。 `consoleintr` 會特別處理 backspace 與其他一些特殊字元

當輸入遇到 newline 時，`consoleintr` 會喚醒等待中的 `consoleread`（如果有的話）。 一旦被喚醒，`consoleread` 就會發現 `cons.buf` 中已經有一整行輸入，接著它會把這行資料複製到 user space，然後透過 system call 的機制將控制權返回給 user space

## 5.2 Code: Console output

對已連接到 console 的 file descriptor 所做的 `write` system call，最終會到達 `uartputc`（[kernel/uart.c:87](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/uart.c#L87)）。 driver 維護了一個輸出緩衝區（`uart_tx_buf`），使得執行寫入的 process 不需要等待 UART 傳送完畢； 相反地，`uartputc` 會將每個字元加入緩衝區，然後呼叫 `uartstart` 來啟動 UART 的傳送（如果尚未開始的話），接著就直接返回。 只有當緩衝區滿了的情況下 `uartputc` 才會停下來等待

每當 UART 傳送完一個 byte，它就會產生一個中斷。 `uartintr` 會呼叫 `uartstart`，這個函式會檢查 UART 是否真的完成傳送，然後把下一個緩衝區中的輸出字元交給 UART 傳送。 因此，如果某個 process 一次寫入多個 byte 到 console，通常第一個 byte 是由 `uartputc` 呼叫 `uartstart` 傳送出去的，其餘在緩衝區裡的 byte 則會由 `uartintr` 在每次中斷到來時持續呼叫 `uartstart` 傳送出去

通常我們會利用「緩衝區」與「中斷」來讓「裝置活動」與「process 活動」解耦。 即使沒有 process 正等著要讀資料，console driver 也能先處理輸入，因此之後的 `read` 呼叫仍然能讀到這些資料。 同樣地，process 也能夠直接輸出資料，而不用等待裝置完成傳送。 這種解耦能提升效能，因為它允許 process 在進行裝置 I/O 的同時繼續執行，而當裝置速度很慢（像 UART）或需要即時反應（像 echo 指定字元）時，這點特別重要。 這種設計理念有時被稱為 I/O 並行（I/O concurrency）

::: tip  
當我們在 shell 中敲下 `a` 鍵，其流程大概如下：

1. shell 內的 `getcmd` 印出 `$` 提示後會呼叫 `gets`
    - 其底層會呼叫 `consoleread`，由於 FIFO 為空，因此會進入 sleep 狀態
2. QEMU 把 `a` 推入 UART FIFO
3. UART 設置 `LSR_RX_READY` bit → PLIC 送 IRQ
4. CPU trap → trap vector → trap handler → `devintr()`
5. `devintr()` → `uartintr()` → 讀 FIFO（透過 `uartgetc()`）
6. `uartintr()` 呼叫 `consoleintr('a')`
7. `consoleintr()` 把 `'a'` 放入 `cons.buf` 內
    - 由於沒有 `'\n'`，所以不會喚醒 `consoleread()`，立即 return
8. 回到被搶佔前正在執行的行程

其中：

- 第 2、3 步都是硬體（QEMU）處理的
- 第三步是要走 `kernelvec` 還是 `uservec`，取決於發生 trap 時 CPU 正在哪個 mode 下
  - 例如如果當時剛好 shell 呼叫了 `read()` 並正睡在 `sleep()` 中，那 CPU 就還處在 kernel space，因此會走 `kernelvec`
  - 而如果 CPU 已經切去另一個 user process 了，那就會走 `uservec`
  - 但不管走哪個，最後都會執行到 `devintr`  
:::

## 5.3 Concurrency in drivers

你可能已經注意到在 `consoleread` 和 `consoleintr` 中有呼叫 `acquire`，這些呼叫會取得鎖，以保護 console driver 的資料結構不被並行存取。 在這裡有三種並行風險：第一是兩個不同 CPU 上的 process 可能同時呼叫 `consoleread`； 第二是硬體可能在某個 CPU 執行 `consoleread` 時觸發 console（實際上是 UART）中斷，打斷該 CPU； 最後是中斷可能發生在另一個 CPU 上，而此時某個 CPU 正在執行 `consoleread`。 第六章會說明如何使用鎖來避免這些情況導致錯誤結果

driver 在處理並行時還有另一個需要注意的情況，那就是某個 process 可能正在等待某個裝置的輸入，但用來通知輸入到達的中斷可能是在另一個 process（或根本沒有 process）執行時產生的。 因此，interrupt handler 無法預期它中斷的 process 或程式碼是什麼，例如 interrupt handler 無法直接使用當前 process 的 page table 去呼叫 `copyout`。 通常 interrupt handler 只會做一些簡單的工作（例如將輸入資料複製到緩衝區），然後喚醒 top-half 的程式碼來完成後續處理

::: tip  
interrupt handler 不能依賴於當前 CPU 所執行的 context，因為它可能與觸發中斷的真正程式無關。 這是因為：

- 中斷可能在任意時間點、任意 CPU 上發生
- 沒有保證當前在執行的就是那個等待輸入的 process。 所以 interrupt handler 常只做 minimal 的事（如複製資料進 buffer），然後喚醒後續能正確處理的程式碼。這就是 top-half / bottom-half 設計的意義  
:::

## 5.4 Timer interrupts

xv6 使用 timer interrupt 來維持系統對「目前時間」的認知，並在多個 compute-bound 的 process 之間進行切換。 timer interrupt 由接在每個 RISC-V CPU 上的時鐘硬體所產生，xv6 會設定每個 CPU 的時鐘硬體，讓它定期對該 CPU 產生中斷

start.c 中的程式碼（[kernel/start.c:53](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/start.c#L53)）會設定一些控制用的 bit，使 supervisor mode 可以存取 timer 控制暫存器，接著會請求第一個 timer interrupt。 `time` 控制暫存器會以固定速率自動遞增，這提供了「目前時間」的概念。 `stimecmp` 暫存器中則存放一個時間點，當 `time` 遞增到該時間點時，CPU 就會產生 timer interrupt。 換句話說，如果將 `stimecmp` 設為 `time` 加上某個值 `x`，就表示在 `x` 個時間單位之後會產生一個 interrupt。 對於 QEMU 的 RISC-V 模擬器來說，1000000 個時間單位大約等於 0.1 秒

timer interrupt 會像其他裝置中斷一樣，經由 `usertrap` 或 `kerneltrap` 和 `devintr` 傳送進來。 timer interrupt 發生時，`scause` 的低位元會被設為 5； trap.c 中的 `devintr` 偵測到這種情況時，會呼叫 `clockintr`（[kernel/trap.c:164](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/trap.c#L164)）。 `clockintr` 會將 ticks 遞增，以讓 kernel 能夠追蹤時間流逝，這個遞增動作只會在其中一個 CPU 上執行，以避免多個 CPU 同時讓時間變快。 `clockintr` 也會喚醒那些正在 `sleep` 系統呼叫中等待的 process，並寫入新的 `stimecmp` 值來排程下一次的 timer interrupt

`devintr` 若遇到 timer interrupt，會回傳 2，以通知 `kerneltrap` 或 `usertrap` 應該呼叫 `yield`，從而讓 CPU 能夠在多個可執行的 process 之間進行切換

kernel 程式碼可能會在執行期間被 timer interrupt 打斷，並因為 `yield` 而進行 context switch，這也是為什麼 `usertrap` 初期的程式碼會小心地先儲存像 `sepc` 這類的狀態，然後才開啟中斷。 這也意味著在撰寫 kernel 程式碼時必須考慮到會有這類 context switch 發生，它可能會在沒有預警的情況下從某個 CPU 被切換到另一個 CPU 上執行

## 5.5 Real world

和許多作業系統一樣，xv6 在 kernel 執行期間也允許中斷發生，甚至允許透過 `yield` 進行 context switch。 這麼做的原因是希望即使在執行一些耗時且複雜的系統呼叫時，也能維持良好的反應速度。 不過，如前所述，在 kernel 中允許中斷也會導入一些額外的複雜性； 因此，有些作業系統選擇只在執行 user code 時才允許中斷

要完整支援一台典型電腦上所有裝置，是一件非常繁瑣的事，因為裝置種類繁多，每個裝置又有很多功能，而且裝置與 driver 之間的通訊協定往往很複雜，甚至缺乏良好文件。 在許多作業系統中，driver 的程式碼量往往超過了 kernel 本身

UART driver 是透過讀取 UART 控制暫存器，每次取回一個 byte 的方式來取得資料的； 因為是由軟體主動驅動資料搬移的，所以這種模式稱為「programmed I/O」。 programmed I/O 實作簡單，但速度太慢，不適合高資料速率的應用。 需要高速搬移大量資料的裝置通常會使用 direct memory access（DMA）的方式，例如現代的硬碟與網路裝置都使用 DMA。 DMA 裝置硬體會直接把接收到的資料寫入 RAM，也會從 RAM 中讀取要傳送的資料。 對於 DMA 裝置而言，driver 會先在 RAM 中準備好資料，然後只需寫一次控制暫存器來告訴裝置處理這些資料即可

當無法預測裝置需要處理的時間點，但其頻率又不太高時，使用中斷是合理的。 但中斷的 CPU 成本很高，因此像網路或硬碟這種高速裝置會使用一些技巧來減少中斷需求。 一種技巧是針對一整批收送請求只產生一次中斷。 另一種技巧則是完全關掉中斷，由 driver 定期檢查裝置是否需要處理，這種技巧稱為 polling。 polling 適用於裝置操作頻率很高的情況，但若裝置大多時間閒置，polling 會浪費大量 CPU 資源。 有些 driver 會根據裝置目前的負載，動態地在 polling 和中斷兩種模式之間切換

UART driver 會先將收到的資料複製到 kernel 的緩衝區，再複製到 user space。 這樣的作法在低資料速率的環境下是合理的，但對於產生或消耗資料速度很快的裝置，這樣的兩次複製會顯著影響效能。 有些作業系統能夠直接在 user-space buffer 和裝置硬體之間搬移資料，這通常需要透過 DMA 來完成

如第一章所述，console 在應用程式看來就像是一般的檔案，應用程式會使用 `read` 和 `write` 系統呼叫來進行輸入與輸出。 不過，有些裝置功能無法透過標準的檔案系統呼叫來表示（例如，開啟或關閉 console driver 的 line buffer）。 Unix 作業系統會提供 `ioctl` 系統呼叫來處理這類情況

有些電腦用途要求系統必須在固定時間內做出反應。 例如在重視安全性的系統中，錯過期限可能會導致災難。 xv6 不適合用在硬即時（hard real-time）的情境，硬即時系統的作業系統通常會以函式庫的形式與應用程式連結，讓系統能分析出最壞情況的反應時間 。xv6 也不適合用於軟即時（soft real-time）應用，也就是偶爾錯過期限可以接受的情境，因為 xv6 的排程器太過簡單，而且有些 kernel 的程式路徑會在長時間內關閉中斷

::: tip  
硬即時系統（例如醫療、飛控）要求絕不能 miss deadline，而軟即時系統（例如影音播放）容許偶爾 miss，但仍要求反應快。 但 xv6 的排程器過於簡單，且某些 kernel 區段會 disable interrupt 太久，這讓它無法保證 deadline，故兩種都不適合  
:::

## 5.6 Exercises

1. 修改 uart.c，讓它完全不使用中斷。 你可能也需要修改 console.c
2. 為網卡新增一個 driver

## Bibliography

- <a id="1">[1]</a>：Martin Michael and Daniel Durich. The NS16550A: UART design and application considerations. http://bitsavers.trailing-edge.com/components/national/_appNotes/AN-0491.pdf, 1987.
