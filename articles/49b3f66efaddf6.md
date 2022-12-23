---
title: "デバッガ自作入門"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに
この記事はEEICアドベントカレンダー2022 24日目の記事として書かれました。前日の記事は[TOOD]()でした！

3年後期実験のどれかをアドカレの記事にしようかなと考えていたのですが、どれもアドカレの記事にしづらい内容でした*1。
そこで、日々お世話になっているツールやソフトウェアを自作する車輪の再発明ネタとして、簡単なデバッガを自作するというのをやってみようと思います。

# デバッガとは
デバッグを支援するソフトウェアを開発する上で欠かせないツールです。
デバッガ自体を使ったことがないという人は本記事よりも先に[VS Codeエディタ入門 Chapter09 デバッグ方法](https://zenn.dev/karaage0703/books/80b6999d429abc8051bb/viewer/898591)などの記事をご覧になることをおすすめします。

プログラムをデバッグする上で大事な要素は **場所**と**状態**であると言えます。

具体的には、場所とはプログラムの中のどの部分を今実行しているのか？という情報です。その粒度は様々で、「関数」「スコープ」「行」「命令」などがあります。場面に応じて実行地点に関する情報を得ることが必要です。
状態とは、ある部分を実行しているときの状況を再現するのに必要な情報全てと言えます。変数の中身、レジスタの中身、スタックの状況、などです。おそらくほとんどの方が、中身を見たい変数を標準(エラー)出力に出してデバッグをする"printfデバッグ"を経験したことがあるはずです。デバッガを使うプログラマは、その箇所でのバグを特定するために、プログラムの表面から何段か低いレイヤの情報(状態)を知ることができます。これにより、「あ、`index`という変数の中身を見てみたら、配列の長さを超えているじゃないか」などの判断が下せるわけです。
まとめると、デバッガとは「あるプログラムの特定の場所での状態を得るツール」と表現できると考えられます。その仕組みを知るために、今回は`ptrace`というLinuxのシステムコールを用いた簡易的なデバッガを自作していきます。

## gdb
代表的なデバッガに[gdb](https://www.sourceware.org/gdb/)があります。使い方を軽く見てみましょう。
gdbに関する詳しい情報は[TODO]などが参考になります。詳しく知りたい方はそちらをご覧ください。
gdbは大きく分けて2つの使い方があります。

### 1: プログラムの特定の箇所にbreakpointを設置して実行
1. -gオプションを付けてコンパイル
```bash
$ gcc -g test.c - test
```
2. 生成したオブジェクトファイルをgdbでdebug
```bash
$ gdb test

GNU gdb (Ubuntu 10.2-0ubuntu1~20.04~1) 10.2
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
...
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Registered pretty printers for UE4 classes
Reading symbols from test...
(No debugging symbols found in test)
(gdb)
```

対話入力に入ります。

3. Breakpointを設定
```bash
b main # main関数のエントリにBreakpointを設定
```

4. 実行
```bash
run <arg> # 引数はここで渡せる
step # 実行を1行ずつ進める
```
設定したbreakpointで実行が一時停止します。

5. 状態を観察する
例えばレジスタの中身を一覧表示するには、特定の地点にいるときに
```bash
(gdb) info registers
rax            0x7fffffffd310      140737488343824
rbx            0x401260            4199008
rcx            0x0                 0
rdx            0x7fffffffd2f0      140737488343792
rsi            0x402004            4202500
rdi            0x402004            4202500
rbp            0x7fffffffd410      0x7fffffffd410
rsp            0x7fffffffd2f0      0x7fffffffd2f0
r8             0x0                 0
r9             0x7ffff7fe0d60      140737354009952
r10            0x402004            4202500
r11            0x7ffff7de7c90      140737351941264
r12            0x401050            4198480
r13            0x7fffffffd500      140737488344320
r14            0x0                 0
r15            0x0                 0
rip            0x7ffff7de7d21      0x7ffff7de7d21 <__printf+145>
eflags         0x246               [ PF ZF IF ]
cs             0x33                51
ss             0x2b                43
ds             0x0                 0
es             0x0                 0
fs             0x0                 0
gs             0x0                 0
```
と実行すれば良いです。

### 2: 既に走っているプロセスにAttachして実行
既に走っているプロセスのデバッグをしたい場面も頻繁にあります。gdbはそのようなユースケースにも対応しています。
1. 対象のプログラムが走っているプロセスのPIDを得る
例えば`test`というプログラムが実行されているプロセスを探すには
```bash
ps | grep test
```
とすればよいです。

2. gdbを起動し、attachする。
```bash
$ gdb

GNU gdb (Ubuntu 10.2-0ubuntu1~20.04~1) 10.2
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
...
(gdb) attach <pid>
```


# 実装方針
上で紹介したgdbの使い方のうち、今回は1.のgdbから直接起動するスタイルのデバッガを作ります。gdbの使い方が掴めたところで、デバッガを実現するための技術について見ていきましょう。

## デバッガを実現する技術: `ptrace`
`ptrace`とは、実行中の他のプロセスの状態を取得したり、状態を改変したりすることができるシステムコールです。これを用いるとgdbやstraceのようなツールが実現できます。まさに、processをtraceするシステムコールです。

gdbの使い方1., 2.,いずれの場合であっても、要するにデバッガがやりたいことは「他のプロセスの状態にRead/Writeする」ことです。この操作は全て`ptrace`というインタフェースに集約されており、具体的な動作は次に説明する"request"によって指定します。

## ptrace request
`ptrace`に具体的にどのような操作をさせるかを定めるものがrequestです。今回の記事で重要となるものにのみ絞って紹介します。興味のある方は`man`コマンドなどで調べてみて下さい。以後、traceされている側を"tracee"、traceする側を"tracer"と表現します。

|Request|機能|
|---|---|
|`PTRACE_PEEKTEXT`| traceeのメモリにおいて、特定のaddressに対応するwordを読み出す。|
|`PTRACE_POKETEXT`| traceeのメモリにおいて、特定のaddressに対応するwordを指定された値に書き換える。|
|`PTRACE_GETREGS` | traceeのレジスタ値のコピーを得る|
|`PTRACE_SETREGS` | traceeのレジスタに特定の値をsetする |
|`PTRACE_CONT`| 停止していたtraceeの実行を再開させる |
|`PTRACE_SINGLESTEP`| 停止しているtraceeの実行を次の命令まで進め、また停止させる。|

PEEK/POKEはそれぞれtraceeのメモリに対するread/write、GETREGS/SETREGSはそれぞれレジスタに対するread/writeに対応しています。また、CONT/SINGLESTEPなどの実行の停止・再開を操作できるRequestも存在します。以上のRequestを使って`ptrace`を叩けば、デバッグしたいプロセスの実行を制御しつつ、状態を見ることができそうです。

## ELF形式
ELF形式とは、LinuxなどのOSSで広く採用されている実行形式です。
最も身近な例を見てみましょう。
```bash
# 何かしらのC言語のソースをコンパイル
$ gcc test.c -o test_elf
$ file test_elf
test_elf: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e4c5302d1da21f7ed6d8148b4c6124172c26a92a, for GNU/Linux 3.2.0, not stripped
```
fileコマンドの出力には、ELF32/64bitのいずれか、Objectのタイプ、アーキテクチャ(X86-64)などの情報が含まれていますね。

### ELFファイルのセグメント
ELFファイルは、その実行バイナリの命令・データを含んでいます。
1. コードセグメント
2. データセグメント

の2つがあります。
コードセグメントにはCPUにより実行される命令列が含まれています(機械語)。
データセグメントには、グローバル変数・static変数が配置されています。実体として存在しているのは初期化されているグローバル変数・static変数だけです。ローカル変数や動的に値が確定していく変数はファイルの中の実体としては存在していません。動的に決まる要素は全てコードセグメントの命令列が表現している、とも言えますね。

デバッガを作る側としては、このセグメントを覗き見たいですよね。ね？

### ELFファイルのヘッダ
ELF形式には3種類のヘッダが存在しており、それぞれがMetadataを保持しています。
[TODO]図

#### ELFヘッダ
先頭に位置する。オブジェクトのタイプ、アーキテクチャなどのMetadataや、他の2つのヘッダにアクセスするのに必要なoffset値を含んでいます。

#### プログラムヘッダ



# 実装してみる
コードは[TODO]にあります。以下では本質部分のみを切り出して、実装過程を説明していきます。

引数

ELFパーサのところ

lookupする

pid

PEEK/POKEでBreakpointをおく

メインループ。止まったら情報表示。

抜ける部分。


# 結果

test_addの例
引数の呼び出し規約から、レジスタの値が引数に一致していることをかく

# おわりに

*1 大規模ソフトウェアはレポートがすでに[記事](TODO)になり、マイクロプロセッサ実験はぱっとしない成果になり、人工知能演習は優勝したがためにまだ内容を世に出せない、という状態でした.
