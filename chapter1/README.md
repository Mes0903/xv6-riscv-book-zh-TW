# Chapter 1：Operating system interfaces

作業系統的任務，是要讓多個程式能夠共享同一部電腦，並提供比硬體本身更多、更加實用的功能。 作業系統負責管理並抽象化底層硬體，使得例如文字處理器這類應用程式無需關心所使用的是哪一種磁碟硬體。 作業系統能讓多個程式共享硬體資源，並使它們能夠同時執行（或至少看起來是同時執行）。 最後，作業系統還提供讓程式之間可以互相溝通的介面，使它們能夠共享資料或協同作業

作業系統透過一組介面來向使用者程式提供服務，但要設計一個好的介面其實並不容易。 一方面，我們希望這個介面要簡單且精確，這樣比較容易實作正確； 但另一方面，我們又會想要提供許多進階的功能給應用程式使用。 解決這個矛盾的訣竅是：設計一組依賴少量機制的介面，並讓這些機制可以組合起來，提供高度的通用性

本書將使用一個具體的作業系統作為例子，來說明作業系統的各種概念。 這個作業系統叫做 xv6，它提供了 Ken Thompson 與 Dennis Ritchie 在 Unix 作業系統中所引入的基本介面，並且模仿了 Unix 的內部設計。 Unix 提供的介面通常「精準但可組合性強」，這讓它意外地擁有很高的通用性。 由於這種介面設計地非常成功，以至於現代的作業系統，例如 BSD、Linux、macOS、Solaris，甚至在某種程度上連 Microsoft Windows 都擁有類 Unix 的介面。 而理解 xv6，是理解這些系統（還有許多其他系統）的一個很好的起點

如圖 1 所示，xv6 採用了傳統的核心（kernel）架構，也就是一個特殊的程式，專門提供執行中程式所需的服務。 每個執行中的程式稱為一個「行程（process）」，它的記憶體會包含指令區、資料區，以及堆疊（stack）。 指令負責實作該程式的運算邏輯； 資料則是程式操作的變數； 堆疊則負責組織與管理程式的函式呼叫。 一台電腦通常會同時擁有許多個行程，但只會有一個核心

![（Figure 1.1: A kernel and two user processes.）](image/os.png)

當一個行程需要呼叫核心的服務時，它會發出一個「系統呼叫」，也就是作業系統介面中的一種呼叫方式。 這個系統呼叫會進入核心，接著核心會執行所請求的服務並返回。 因此一個行程的執行會在使用者空間與核心空間之間交替進行

如後續章節將詳細說明的，核心會使用 CPU 提供的硬體保護機制（本書使用 CPU 一詞來指稱執行運算的硬體元件； 其他文件，如 RISC-V 規格，會使用 processor、core 或 hart 等詞來代替 CPU），來確保每個在使用者空間中執行的行程只能存取自己的記憶體。 核心本身會於具備特權的硬體模式執行，以實作這些保護機制； 而使用者程式則在沒有這些特權的情況下執行。 當一個使用者程式發出系統呼叫時，硬體會提升執行權限，並開始執行核心中事先安排好的函式

使用者程式所能看見的介面由核心提供的所有系統呼叫組成。 xv6 核心提供了一部分傳統 Unix 核心所具備的服務與系統呼叫。 圖 1.2 列出了 xv6 所提供的全部系統呼叫：

<span class = "center-column">

| **System call**                               | **Description**                                                                 |
|----------------------------------------------|---------------------------------------------------------------------------------|
| `int fork()`                                  | Create a process, return child's PID.                                           |
| `int exit(int status)`                        | Terminate the current process; status reported to wait(). No return.           |
| `int wait(int *status)`                       | Wait for a child to exit; exit status in *status; returns child PID.           |
| `int kill(int pid)`                           | Terminate process PID. Returns 0, or -1 for error.                              |
| `int getpid()`                                | Return the current process's PID.                                               |
| `int sleep(int n)`                            | Pause for n clock ticks.                                                        |
| `int exec(char *file, char *argv[])`          | Load a file and execute it with arguments; only returns if error.              |
| `char *sbrk(int n)`                           | Grow process's memory by n zero bytes. Returns start of new memory.            |
| `int open(char *file, int flags)`             | Open a file; flags indicate read/write; returns an fd (file descriptor).       |
| `int write(int fd, char *buf, int n)`         | Write n bytes from buf to file descriptor fd; returns n.                        |
| `int read(int fd, char *buf, int n)`          | Read n bytes into buf; returns number read; or 0 if end of file.               |
| `int close(int fd)`                           | Release open file fd.                                                           |
| `int dup(int fd)`                             | Return a new file descriptor referring to the same file as fd.                 |
| `int pipe(int p[])`                           | Create a pipe, put read/write file descriptors in p[0] and p[1].               |
| `int chdir(char *dir)`                        | Change the current directory.                                                   |
| `int mkdir(char *dir)`                        | Create a new directory.                                                         |
| `int mknod(char *file, int, int)`             | Create a device file.                                                           |
| `int fstat(int fd, struct stat *st)`          | Place info about an open file into *st.                                         |
| `int link(char *file1, char *file2)`          | Create another name (file2) for the file file1.                                 |
| `int unlink(char *file)`                      | Remove a file.                                                                  |

（Figure 1.2: Xv6 system calls. If not otherwise stated, these calls return 0 for no error, and -1 if there’s an error）

</span>

本章接下來將粗略地介紹 xv6 所提供的幾項服務，包含行程管理、記憶體、檔案描述符（file descriptors）、管道（pipes），以及檔案系統，並透過程式碼範例與說明，來展示 Unix 的命令列介面 shell 是如何使用這些功能。 從 shell 對系統呼叫的使用方式，可以看出這些呼叫是如何被精心設計的

shell 是一個普通的程式，它負責讀取使用者輸入的指令並執行。 其為一個使用者程式，而不是核心的一部分，這點凸顯了系統呼叫介面的強大之處：shell 並不是什麼特別的程式。 這也代表 shell 很容易被替換； 因此，現代的 Unix 系統都有各式各樣的 shell 可供選擇，每種 shell 都有自己獨特的使用者介面與腳本功能。 xv6 的 shell 是 Unix Bourne shell 精神的一個簡單實作，其程式碼可以在 `user/sh.c:1` 內找到

## 1.1 Processes and memory

一個 xv6 行程由使用者空間中的記憶體（包含指令、資料與堆疊）以及屬於該行程、只有核心能存取的內部狀態所構成。 xv6 採用時間分割（time-sharing）的方式來管理行程：它會在等待執行的行程之間，自動切換可用的 CPU。 當某個行程暫停執行時，xv6 會儲存該行程的 CPU 暫存器，等下次執行該行程時再將其還原。 核心還會為每個行程分配一個被稱為 PID（process identifier，行程識別碼）的編號

一個行程可以透過 `fork` 系統呼叫來建立一個新的行程。 `fork` 會將原本呼叫者的記憶體完整複製給新建立的行程：它會將呼叫者的指令、資料與堆疊全部複製到新行程中。 `fork` 會在原本與新建立的行程中各自返回一次，在原本的行程中，`fork` 會返回新行程的 PID； 而在新建立的行程中，`fork` 則返回 0。 原本的行程與新建立的行程，通常分別被稱為「父行程」與「子行程」

舉例來說，請看以下這段以 C 語言撰寫的程式碼片段：

```c
int pid = fork();
if(pid > 0){
  printf("parent: child=%d\n", pid);
  pid = wait((int *) 0);
  printf("child %d is done\n", pid);
} else if(pid == 0){
  printf("child: exiting\n");
  exit(0);
} else {
  printf("fork error\n");
}
```

`exit` 系統呼叫會讓呼叫它的行程停止執行，並釋放像是記憶體與已開啟檔案這類資源。 `exit` 接收一個整數作為狀態參數，慣例上，0 表示成功，1 表示失敗

`wait` 系統呼叫會回傳一個已結束（或被終止）的子行程的 PID，並將該子行程的結束狀態寫入「傳給 `wait` 的記憶體位置」； 如果目前還沒有任何已結束的子行程，`wait` 就會阻塞，直到有一個子行程結束。 如果呼叫者沒有子行程，`wait` 會立即回傳 -1。 如果父行程不在意子行程的結束狀態，它可以傳入 0 當作 `wait` 的參數

在這個範例中，這兩行輸出：

```
parent: child=1234 
child: exiting
```

的順序可能會互換（甚至交錯），這取決於父行程與子行程誰先執行到 `printf`。 當子行程結束後，父行程中的 `wait` 呼叫會返回，接著父行程會印出 `parent: child 1234 is done`。 雖然子行程最初擁有與父行程相同的記憶體內容，但父子行程各自擁有獨立的記憶體與暫存器，因此在其中一方改變變數時，不會影響到另一方。 例如，當 `wait` 的返回值被存入父行程的變數 `pid` 中時，也不會改變子行程中的 `pid` 變數，子行程中的 `pid` 值仍然是 0

`exec` 系統呼叫會用從檔案系統中載入的程式映像（memory image），取代呼叫該行程原本的記憶體內容。 這個檔案必須具有特定格式，格式中會定義檔案的哪個部分是指令、哪個部分是資料、從哪個指令開始執行等等。 xv6 採用 ELF 格式，這部分會在第三章中進一步說明，通常這個檔案是將原始碼編譯後所產生的結果。 當 `exec` 成功時，它不會返回到呼叫它的程式； 相反地，從檔案中載入的指令會從 ELF 標頭中指定的進入點開始執行。 `exec` 接收兩個參數：一是包含可執行檔的檔案名稱，另一是字串陣列作為參數

舉例來說：

```c
char *argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;
exec("/bin/echo", argv);
printf("exec error\n");
```

這段程式碼會將目前的程式取代為 `/bin/echo` 這個程式的執行實例，並帶入參數列表 `echo` 與 `hello`。 大多數程式會忽略參數陣列的第一個元素，這個元素慣例上是程式本身的名稱

xv6 的 shell 使用上述系統呼叫來代替使用者執行程式。 這個 shell 的主體結構相當簡單； 可以參考 [`user/sh.c:146`](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L146) 中的 `main` 函式。 主迴圈會透過 `getcmd` 從使用者那讀取一行輸入。 接著它會呼叫 `fork`，建立 shell 行程的複本。 父行程會呼叫 `wait`，而子行程則負責執行命令

例如，如果使用者在 shell 中輸入 `"echo hello"`，`runcmd` 就會被呼叫，並以 `"echo hello"` 作為參數。 `runcmd`（參考 [`user/sh.c:55`](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L55)）會執行真正的命令。 對於 `"echo hello"`，它會呼叫 `exec`（見 [`user/sh.c:79`](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L79)）。 如果 `exec` 成功，子行程就會開始執行 `echo` 的指令，而不是 `runcmd`。 之後的某個時間點，`echo` 會呼叫 `exit`，此時父行程中的 `wait` 會返回，控制流程便會回到 `main` 函式中（見 `user/sh.c:146`）

你可能會想，為什麼 `fork` 和 `exec` 不直接合併成一個呼叫； 我們稍後會看到，shell 透過分離它們來實作輸入/輸出的重導（redirection）功能。 為了避免「建立一個行程複本、接著馬上被 `exec` 替換掉」這種浪費，作業系統核心會針對這類用途對 `fork` 的實作進行最佳化，例如採用虛擬記憶體技術中的 copy-on-write（詳見第 4.6 節）

xv6 對於大部分使用者空間的記憶體配置是隱式進行的：`fork` 會配置足夠的記憶體來複製父行程的記憶體給子行程，`exec` 則會配置足夠的記憶體以容納可執行檔的內容。 若某個行程在執行期間需要更多記憶體（例如 `malloc`），可以呼叫 `sbrk(n)` 來將其資料區延伸 `n` 個 0 位元組； `sbrk` 會回傳新記憶體的起始位置

## 1.2 I/O and File descriptors

檔案描述符（File descriptors）是一個較小的整數，用來表示一個由核心管理的物件，行程可以從這個物件讀取或寫入資料。 行程可以透過開啟檔案、目錄、或裝置、建立管道，或複製現有的描述符，來取得一個檔案描述符。 為了簡化說明，我們會將檔案描述符所指向的物件統稱為「檔案」； 檔案描述符這個介面抽象化了檔案、管道與裝置之間的差異，使它們看起來都像是位元組流（byte stream）。 我們會把輸入與輸出稱為 I/O

::: tip  
雖然名字叫「file」，但其實它可以指向任何 I/O 來源，例如檔案、終端機、管道、裝置等。 它是一個抽象層，讓程式不用知道背後實體是什麼，就能進行讀寫操作  
:::

在內部，xv6 核心使用檔案描述符作為每個行程表格中的索引，因此每個行程都有一個從 0 開始的私有檔案描述符空間。 依照慣例，行程從描述符 0（標準輸入）讀取，將輸出寫到描述符 1（標準輸出），將錯誤訊息寫到描述符 2（標準錯誤）。 如我們後面會看到，shell 利用這些慣例來實作 I/O 重導與管線。 預設情況下這三個描述符對應到主控台，因此 shell 會確保它總是打開這三個檔案描述符

`read` 與 `write` 系統呼叫會根據檔案描述符，從已開啟的檔案中讀取或寫入位元組。 呼叫 `read(fd, buf, n)` 會從檔案描述符 `fd` 所指向的檔案中最多讀取 `n` 個位元組，接著將它們複製到 `buf` 中，並回傳實際讀取的位元組數。 每個指向檔案的檔案描述符都會有一個相關的偏移量（offset）。 `read` 會從目前的偏移位置讀取資料，然後將偏移量往後推移「該次讀取的位元組數」，下一次 `read` 會接續讀取後面的資料。 當沒有更多資料可讀時，`read` 會回傳 0，以表示檔案結尾

`write(fd, buf, n)` 這個呼叫會將 `buf` 中的 `n` 個位元組寫入檔案描述符 `fd` 所指向的目標，並回傳實際寫入的位元組數。 只有在發生錯誤時，才可能寫入少於 `n` 個位元組。 與 `read` 類似，`write` 會從目前的檔案偏移位置開始寫入資料，並在寫入後將偏移量增加「該次寫入的位元組數」，每次 `write` 都會從上一次結束的位置繼續

以下這段程式碼（它構成了 `cat` 程式的核心邏輯）會將資料從標準輸入複製到標準輸出。 如果發生錯誤，它會將錯誤訊息輸出到標準錯誤：

```c
char buf[512];
int n;

for(;;){
  n = read(0, buf, sizeof buf);
  if(n == 0)
    break;
  if(n < 0){
    fprintf(2, "read error\n");
    exit(1);
  }
  if(write(1, buf, n) != n){
    fprintf(2, "write error\n");
    exit(1);
  }
}
```

這段程式碼中最值得注意的是，`cat` 並不知道它是從檔案、主控台，還是管道中讀取資料。 同樣地，`cat` 也不知道它是把資料印到主控台、寫到檔案，還是其他地方。 檔案描述符的使用方式，加上將描述符 0 視為輸入、描述符 1 視為輸出的慣例，使得 `cat` 的實作可以非常簡潔

`close` 系統呼叫會釋放一個檔案描述符，讓它可以被日後的 `open`、`pipe` 或 `dup` 系統呼叫（下文會說明）重新使用。 新分配的檔案描述符總是從目前行程中尚未使用的最小編號開始

檔案描述符與 `fork` 的互動，使得 I/O 重導的實作變得簡單。 `fork` 會連同父行程的記憶體一起複製其檔案描述符表，因此子行程啟動時會擁有與父行程完全相同的已開啟檔案。 系統呼叫 `exec` 雖然會取代呼叫者的記憶體，但會保留它的檔案描述符表。 這樣的行為允許 shell 透過「先 `fork` 出子行程、在子行程中重新開啟指定的檔案描述符、再呼叫 `exec` 來執行新程式」的方式來實作 I/O 重導

以下是一段簡化版本的 shell 程式碼，模擬執行 `cat < input.txt` 這條命令的行為：

```c
char *argv[2];

argv[0] = "cat";
argv[1] = 0;
if(fork() == 0) {
  close(0);
  open("input.txt", O_RDONLY);
  exec("cat", argv);
}
```

當子行程關閉檔案描述符 0 之後，`open` 一定會將新開啟的 `input.txt` 指派到這個描述符，因為 0 是當前可用的最小檔案描述符。 接下來 `cat` 的執行中，檔案描述符 0（標準輸入）就會對應到 `input.txt`。 這整個過程只改變了子行程的描述符，父行程的檔案描述符不會受到影響

xv6 shell 中的 I/O 重導就是依照這個方式運作的（[`user/sh.c:83`](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L83)）。 請記得，在程式執行到這個階段時，shell 已經建立了子行程，而 `runcmd` 會接著呼叫 `exec` 來載入新的程式

`open` 的第二個參數是一組用位元表示的旗標，用來控制 `open` 的行為。 這些可用的旗標定義在 fcntl（file control）標頭檔中（[`kernel/fcntl.h:1-5`](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/fcntl.h#L1-L5)），包括：`O_RDONLY`、`O_WRONLY`、`O_RDWR`、`O_CREATE` 與 `O_TRUNC`，它們分別代表以讀取、寫入、讀寫模式開啟檔案，若檔案不存在則建立，或將檔案長度截斷為零

現在你應該可以理解，為什麼將 `fork` 和 `exec` 設計為分開的呼叫是有幫助的：這兩者之間的空檔，讓 shell 有機會在不影響主 shell 的情況下，重新設定子行程的 I/O。 如果想像一個虛構的 `forkexec` 系統呼叫，那麼要在這樣的架構下實作 I/O 重導會變得相當麻煩。 可能的做法包括：shell 在呼叫 `forkexec` 前修改自己的 I/O 設定（然後再復原）； 或者讓 `forkexec` 接收 I/O 重導的參數； 或者（這是最不理想的）讓每個像 `cat` 這樣的程式都學會自己做 I/O 重導

雖然 `fork` 會複製檔案描述符表，但每個底層檔案的偏移量仍然會在父子行程之間共用。 請看以下這段程式碼：

```c
if(fork() == 0) {
  write(1, "hello ", 6);
  exit(0);
} else {
  wait(0);
  write(1, "world\n", 6);
}
```

在這段程式執行結束後，與檔案描述符 1 相關聯的檔案中，將會包含 `hello world` 這段資料。 父行程的 `write`（由於有 `wait` 的關係，會在子行程執行完畢後才執行）會從子行程 `write` 所留下的位置繼續寫入。 這種行為有助於讓 shell 指令序列產生連續的輸出，例如 `(echo hello; echo world) >output.txt`

`dup` 系統呼叫會複製一個已存在的檔案描述符，並回傳一個新的描述符，這個新的描述符會指向相同的底層 I/O 物件。 這兩個檔案描述符會共用一個偏移量，就像透過 `fork` 複製出的描述符一樣。 以下是另一種寫入 `hello world` 到檔案的方法：

```c
fd = dup(1);
write(1, "hello ", 6);
write(fd, "world\n", 6);
```

如果兩個檔案描述符是透過一連串的 `fork` 和 `dup` 呼叫，從同一個原始描述符衍生而來，那它們就會共用偏移量。 否則，即使它們是對同一個檔案使用 `open` 所得到的描述符，也不會共享偏移量。 `dup` 允許 shell 實作像這樣的指令：`ls existing-file non-existing-file > tmp1 2>&1`。 其中的 `2>&1` 是告訴 shell，將描述符 2（標準錯誤）設為描述符 1（標準輸出）的副本。 這樣，現有檔案的檔名與不存在檔案的錯誤訊息，會同時寫入到 `tmp1` 這個檔案中。 雖然 xv6 的 shell 不支援錯誤輸出的 I/O 重導，但你現在已經知道如何實作它了

檔案描述符是一個強大的抽象，因為它隱藏了它所連接對象的細節：一個行程在寫入檔案描述符 1 時，實際上可能是在寫入一個檔案、一個像主控台這樣的裝置，或是一個管道

## 1.3 Pipes

管道（pipe）是一塊由核心管理的小型緩衝區，對行程而言，它表現為一對檔案描述符：一個用於讀取、一個用於寫入。 將資料寫入管道的一端，會使這些資料可以從管道的另一端被讀取。 管道提供了一種讓行程之間可以互相溝通的方式

以下這段範例程式碼執行了 `wc` 這個程式，並將其標準輸入接到一個管道的讀取端：

```c
int p[2];
char *argv[2];

argv[0] = "wc";
argv[1] = 0;

pipe(p);
if(fork() == 0) {
  close(0);
  dup(p[0]);
  close(p[0]);
  close(p[1]);
  exec("/bin/wc", argv);
} else {
  close(p[0]);
  write(p[1], "hello world\n", 12);
  close(p[1]);
}
```

上例程式呼叫了 `pipe`，這會建立一條新的管道，並將讀取與寫入的檔案描述符存放在陣列 `p` 中。 在 `fork` 之後，父行程與子行程都擁有指向該管道的描述符。 子行程會呼叫 `close` 與 `dup`，讓描述符 0 指向管道的讀取端，然後關閉 `p` 中的描述符，接著呼叫 `exec` 來執行 `wc`。 當 `wc` 從標準輸入讀取時，它實際上是從這條管道讀資料。 父行程會關閉管道的讀取端、寫入資料，然後再關閉寫入端

如果沒有資料可讀，則對管道進行 `read` 時會等待「資料被寫入」或「所有指向寫入端的描述符都已關閉」這兩種情況發生； 在後者情況下，`read` 會回傳 0，就像讀到檔案結尾一樣。 `read` 會阻塞直到確定不會有新資料出現，這正是為什麼子行程在執行 `wc` 之前一定要關閉管道寫入端的原因之一：如果 `wc` 有一個描述符仍然指向管道的寫入端，那它就永遠看不到檔案結尾（EOF）

xv6 的 shell 會用類似上述程式碼的方式來實作像 `grep fork sh.c | wc -l` 這類管線指令。 子行程會建立一條管道，用來連接管線左端與右端。 接著，它會為管線左端呼叫 `fork` 與 `runcmd`，接著為右端也呼叫 `fork` 與 `runcmd`，並等待兩者完成。 管線的右端可能本身也包含管線（例如 `a | b | c`），這會額外再 `fork` 出兩個子行程（分別給 `b` 和 `c`）。 因此，shell 最終可能建立出一棵行程樹。 這棵樹的葉節點是實際執行的命令，而內部節點則是負責等待左右子行程完成的過渡行程

管道在表面上看起來與暫存檔案並無太大差別，這段管線程式

```sh
echo hello world | wc  
```

也可以改用暫存檔來實作，不用管道：

```sh
echo hello world >/tmp/xyz; wc </tmp/xyz  
```

但在這種情況下，管道相較於暫存檔至少有三項優勢：

- 第一，管道會自動清除自身； 使用檔案重導時，執行完後 shell 必須另外小心地移除 `/tmp/xyz`
- 第二，管道能傳遞任意長度的資料流，而檔案重導方式則需要硬碟上有足夠空間來儲存全部資料
- 第三，管道允許管線中的各階段並行執行，而使用檔案的做法則要求第一個程式必須先結束，第二個才能開始
