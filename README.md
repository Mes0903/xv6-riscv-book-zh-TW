---
title: xv6 riscv book 繁體中文翻譯
date: 2025-08-04
tag: 
- OS
- risc-v
category: 
- OS
- risc-v
star: true
---

# xv6 riscv book 繁體中文翻譯

這是 xv6 riscv book 的繁體中文翻譯，順便加了一些我自己的筆記，由於是放在 Blog 上給我自己看的，所以對原文有做一些使語意更加順暢的小修改，還加上了給我 Blog 使用的標籤

- [本翻譯的 repo 連結](https://github.com/Mes0903/xv6-riscv-book-zh-TW)
- [原 repo 的連結](https://github.com/mit-pdos/xv6-riscv-book/tree/xv6-riscv)
  - 翻譯本文時以 commit e2d964e 為主
  - 原 repo 是以 latex 寫的，但我想要擺在我的 blog 上所以就一如往常的寫成 markdown 了，需要 latex 的話可能可以照著[這個 repo](https://github.com/HelloYJohn/xv6-riscv-book-zh-cn) 去改一下

原文用於 MIT 6.828 及 6.1810 的課堂教學中，共有 1~10 章，最後一章為非常短的 Summary 所以我就沒翻了

最後是 Blog 上的文章連結，畢竟是以放在 Blog 上為前提寫的，因此也會比較好閱讀

- [blog link](https://mes0903.github.io/OS/xv6-riscv-book-zh-TW/)

## LICENSE

LICENSE 遵循原 repo 使用 MIT：

The xv6 book sources are:

Copyright (c) 2006-2024 Russ Cox, Frans Kaashoek, and Robert Morris,
                        Massachusetts Institute of Technology

Permission is hereby granted, free of charge, to any person obtaining a copy of
this source and associated documentation files (the "Book"), to deal in the Book
without restriction, including without limitation the rights to use, copy,
modify, merge, publish, distribute, sublicense, and/or sell copies of the Book,
and to permit persons to whom the Book is furnished to do so, subject to the
following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Book.

THE BOOK IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED,
INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A
PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE BOOK OR THE USE OR OTHER DEALINGS IN THE BOOK.

## Foreword and acknowledgments

以下為原文的 Foreword and acknowledgments：

This is a draft text intended for a class on operating systems. It explains the main concepts of operating systems by studying an example kernel, named xv6. Xv6 is modeled on Dennis Ritchie’s and Ken Thompson’s Unix Version 6 (v6)<sup>[[1]](#1)</sup>. Xv6 loosely follows the structure and style of v6, but is implemented in ANSI C<sup>[[2]](#2)</sup> for a multi-core RISC-V<sup>[[3]](#3)</sup>.

This text should be read along with the source code for xv6, an approach inspired by John Lions’ Commentary on UNIX 6th Edition<sup>[[4]](#4)</sup>; the text has hyperlinks to the source code at https://github.com/mit-pdos/xv6-riscv. See https://pdos.csail.mit.edu/6.1810 for additional pointers to on-line resources for v6 and xv6, including several lab assignments using xv6.

We have used this text in 6.828 and 6.1810, the operating system classes at MIT. We thank the faculty, teaching assistants, and students of those classes who have all directly or indirectly contributed to xv6. In particular, we would like to thank Adam Belay, Austin Clements, and Nickolai Zeldovich. Finally, we would like to thank people who emailed us bugs in the text or suggestions for improvements: Abutalib Aghayev, Sebastian Boehm, brandb97, Anton Burtsev, Raphael Carvalho, Tej Chajed,Brendan Davidson, Rasit Eskicioglu, Color Fuzzy, Wojciech Gac, Giuseppe, Tao Guo, Haibo Hao, Naoki Hayama, Chris Henderson, Robert Hilderman, Eden Hochbaum, Wolfgang Keller, Paweł Kraszewski, Henry Laih, Jin Li, Austin Liew, lyazj@github.com, Pavan Maddamsetti, Jacek Masiulaniec, Michael McConville, m3hm00d, miguelgvieira, Mark Morrissey, Muhammed Mourad, Harry Pan, Harry Porter, Siyuan Qian, Zhefeng Qiao, Askar Safin, Salman Shah, Huang Sha, Vikram Shenoy, Adeodato Simó, Ruslan Savchenko, Pawel Szczurko, Warren Toomey, tyfkda, tzerbib, Vanush Vaswani, Xi Wang, and Zou Chang Wei, Sam Whitlock, Qiongsi Wu, LucyShawYang, ykf1114@gmail.com, and Meng Zhou

If you spot errors or have suggestions for improvement, please send email to Frans Kaashoek and Robert Morris (kaashoek,rtm@csail.mit.edu).

## Bibliography

- <a id="1">[1]</a>：Dennis M. Ritchie and Ken Thompson. The UNIX time-sharing system. Commun. ACM, 17(7):365–375, July 1974.
- <a id="2">[2]</a>：Brian W. Kernighan. The C Programming Language. Prentice Hall Professional Technical Reference, 2nd edition, 1988.
- <a id="3">[3]</a>：David Patterson and Andrew Waterman. The RISC-V Reader: an open architecture Atlas. Strawberry Canyon, 2017.
- <a id="4">[4]</a>：John Lions. Commentary on UNIX 6th Edition. Peer to Peer Communications, 2000.
