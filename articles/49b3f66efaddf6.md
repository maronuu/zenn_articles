---
title: "デバッガを自作してみよう"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

この記事は[EEIC Advent Calendar 2022](https://adventar.org/calendars/7892) 24日目の記事です。EEICとは東京大学工学部電子情報工学科・電気電子工学科を指します。 

# はじめに

こんにちは、eeic2022のいるんごです。12/24の担当だったのですが、色々とタスクや予定が重なり投稿が遅れてしまいました、すみません...。
3年後期実験のどれかをアドカレの記事にしようかなと考えていたのですが、どれもアドカレの記事にしづらい内容でした[^1]。
そこで、日々お世話になっているツールやソフトウェアを自作する車輪の再発明ネタとして、簡単なデバッガを自作するというのをやってみようと思います。

完成品のコードは[こちら](https://github.com/maronuu/SimpleDebugger)

https://github.com/maronuu/SimpleDebugger

完成した自作デバッガの動作は次のようなものです:
![](https://storage.googleapis.com/zenn-user-upload/3777ba9e8443-20221226.gif)

# デバッガとは
デバッグを支援するソフトウェアを開発する上で欠かせないツールです。
デバッガ自体を使ったことがないという人は本記事よりも先に[VS Codeエディタ入門 Chapter09 デバッグ方法](https://zenn.dev/karaage0703/books/80b6999d429abc8051bb/viewer/898591)などの記事をご覧になることをおすすめします。

プログラムをデバッグする上で大事な要素は **場所**と**状態**であると言えます。

具体的には、場所とはプログラムの中のどの部分を今実行しているのか？という情報です。その粒度は様々で、「関数」「スコープ」「行」「命令」などがあります。場面に応じて実行地点に関する適切な情報を得ることが必要です。
状態とは、ある部分を実行しているときの状況を再現するのに必要な情報全てと言えます。変数の中身、レジスタの中身、スタックの状況、などです。おそらくほとんどの方が、中身を見たい変数を標準(エラー)出力に出してデバッグをする"printfデバッグ"を経験したことがあるはずです。デバッガを使うプログラマは、その箇所でのバグを特定するために、プログラムの表面から何段か低いレイヤの情報(状態)を知ることができます。これにより、「あ、`index`という変数の中身を見てみたら、配列の長さを超えているじゃないか」などの判断が下せるわけです。
まとめると、デバッガとは「あるプログラムの特定の場所での状態を得るツール」と表現できると考えられます。その仕組みを知るために、今回は`ptrace`というLinuxのシステムコールを用いた簡易的なデバッガを自作していきます。

## gdb
代表的なデバッガに[gdb](https://www.sourceware.org/gdb/)があります。使い方を軽く見てみましょう。
gdbに関する詳しい情報は[この記事](https://qiita.com/arene-calix/items/a08363db88f21c81d351)などが参考になります。詳しく知りたい方はそちらをご覧ください。
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
|`PTRACE_TRACEME`|このプロセスが親プロセスにtraceされるという関係を明示する。|
|`PTRACE_PEEKTEXT`| traceeのメモリにおいて、特定のaddressに対応するwordを読み出す。|
|`PTRACE_POKETEXT`| traceeのメモリにおいて、特定のaddressに対応するwordを指定された値に書き換える。|
|`PTRACE_GETREGS` | traceeのレジスタ値のコピーを得る|
|`PTRACE_SETREGS` | traceeのレジスタに特定の値をsetする |
|`PTRACE_CONT`| 停止していたtraceeの実行を再開させる |
|`PTRACE_SINGLESTEP`| 停止しているtraceeの実行を次の命令まで進め、また停止させる。|

PEEK/POKEはそれぞれtraceeのメモリに対するread/write、GETREGS/SETREGSはそれぞれレジスタに対するread/writeに対応しています。また、CONT/SINGLESTEPなどの実行の停止・再開を操作できるRequestも存在します。以上のRequestを使って`ptrace`を叩けば、デバッグしたいプロセスの実行を制御しつつ、状態を見ることができそうです。

## ELF形式
ELF形式とは、LinuxなどのOSで広く採用されている実行形式です。
今回作成するデバッガはgdbなどのデバッガと同様にELF形式の実行可能ファイルを対象にすることとします。必要最低限のELFの知識を確認しておきましょう。

最も身近な例として、gccでC言語のソースをコンパイルしてみましょう。
```bash
# 何かしらのC言語のソースをコンパイル
$ gcc test.c -o test_elf
$ file test_elf
test_elf: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e4c5302d1da21f7ed6d8148b4c6124172c26a92a, for GNU/Linux 3.2.0, not stripped
```
コンパイルして得られた実行可能ファイルはELF形式です。`file`コマンドの出力には、ELF32/64bitのいずれか、Objectのタイプ、アーキテクチャ(X86-64)などの情報が含まれていますね。

### ELFファイルの構造
ELFファイルはざっくりと分けてヘッダ・コード領域・データ領域に分けることが出来ます。
コード領域は実行される命令列が機械語として格納されています。
データ領域には、グローバル変数やStatic変数などが格納されています。
ローカル変数やその他動的に定まる変数は実行時にメモリのスタック領域やヒープ領域に確保されるため、ファイルには含まれていません。
デバッガを作るには、実行を停止したい関数のSymbol名をデータ領域から引き、コード領域の命令列の中から関数実行に対応する命令を特定できれば良いということが見えてきました。

### ELFファイルのヘッダ
これらの領域に適切にアクセスするためのMetadataは3種類のヘッダが保持しています。

- **ELFヘッダ** (`Elf64_Ehdr`)
先頭に位置するヘッダです。オブジェクトのタイプ、アーキテクチャなどのMetadataや、他の2つのヘッダにアクセスするのに必要なoffset値を含んでいます。Cでは以下のようなメンバをもつ構造体として実装されています。
```c
typedef struct {
    unsigned char   e_ident[EI_NIDENT];   /* Id bytes */
    Elf64_Quarter   e_type;               /* file type */
    Elf64_Quarter   e_machine;            /* machine type */
    Elf64_Half      e_version;            /* version number */
    Elf64_Addr      e_entry;              /* entry point */
    Elf64_Off       e_phoff;              /* Program hdr offset */
    Elf64_Off       e_shoff;              /* Section hdr offset */
    Elf64_Half      e_flags;              /* Processor flags */
    Elf64_Quarter   e_ehsize;             /* sizeof ehdr */
    Elf64_Quarter   e_phentsize;          /* Program header entry size */
    Elf64_Quarter   e_phnum;              /* Number of program headers */
    Elf64_Quarter   e_shentsize;          /* Section header entry size */
    Elf64_Quarter   e_shnum;              /* Number of section headers */
    Elf64_Quarter   e_shstrndx;           /* String table index */
} Elf64_Ehdr;
```

- **プログラムヘッダ** (`Elf64_Phdr`)
セグメントの情報が含まれているヘッダです。1つのセグメントに対して1つ存在しています。ファイル上のセグメントがどのような属性でどこに読み込まれるのかという情報を保持しています。今回はメインではないため詳細は割愛します。

- **セクションヘッダ** (`Elf64_Shdr`)
セクションヘッダは各セクションの位置、サイズ、link情報などを保持しています。1セクションに対して1つ存在していて、セクションヘッダの集合がセクションヘッダテーブルとして管理されています。ヘッダ単体の構造体としての定義は次のようになります。
```c
typedef struct {
    Elf64_Half      sh_name;        /* section name */
    Elf64_Half      sh_type;        /* section type */
    Elf64_Xword     sh_flags;       /* section flags */
    Elf64_Addr      sh_addr;        /* virtual address */
    Elf64_Off       sh_offset;      /* file offset */
    Elf64_Xword     sh_size;        /* section size */
    Elf64_Half      sh_link;        /* link to another */
    Elf64_Half      sh_info;        /* misc info */
    Elf64_Xword     sh_addralign;   /* memory alignment */
    Elf64_Xword     sh_entsize;     /* table entry size */
} Elf64_Shdr;    
```


# 実装してみる
それでは、これまで学んできた`ptrace`やELFの知識を使って実装していきましょう。

以下では本質部分のみを切り出して、実装過程を説明していきます。エラーハンドリングなどは記事では省略している部分も多いです。完全なコードを見たい方は[リポジトリ](https://github.com/maronuu/SimpleDebugger)をどうぞ。


## ElfHandlerの定義
ELFファイルの中身やヘッダ、Traceに関する情報をまとめて扱うHandler構造体を定義します。
```c
#include <elf.h>
#include <sys/user.h>
...

typedef struct ElfHandler {
    Elf64_Ehdr *ehdr;               // ELF header
    Elf64_Phdr *phdr;               // program header
    Elf64_Shdr *shdr;               // section header
    uint8_t *mem;                   // memory map of the executable
    char *exec_cmd;                 // exec command
    char *symbol_name;              // symbol name to be traced
    Elf64_Addr symbol_addr;         // symbol address
    struct user_regs_struct regs;   // registers
} ElfHandler_t;
```
`user_regs_struct`という構造体は`user.h`というヘッダで定義されている構造体で、レジスタの各番号の名前と値がメンバとして定義されています。必要なメンバのみを抜き出すと以下のようになっています(X86_64)。
```c
struct user_regs_struct {
    unsigned long long int r15;
    unsigned long long int r14;
    unsigned long long int r13;
    unsigned long long int r12;
    unsigned long long int rbp;
    unsigned long long int rbx;
    unsigned long long int r11;
    unsigned long long int r10;
    unsigned long long int r9;
    unsigned long long int r8;
    unsigned long long int rax;
    unsigned long long int rcx;
    unsigned long long int rdx;
    unsigned long long int rsi;
    unsigned long long int rdi;
    unsigned long long int orig_rax;
    unsigned long long int rip;
    unsigned long long int cs;
    unsigned long long int eflags;
    unsigned long long int rsp;
    unsigned long long int ss;
    unsigned long long int fs_base;
    unsigned long long int gs_base;
    unsigned long long int ds;
    unsigned long long int es;
    unsigned long long int fs;
    unsigned long long int gs;
};
```

## 引数のParse
デバッガの実行は`./debugger <executable> <symbol_name>`という形式とします。例えば`test.c`というプログラムの`say_hello`というsymbol名の関数にBreakpointを置くには、gccでコンパイルした後、
```c
$ ./debugger ./test say_hello
```
と実行します。
コマンドライン引数から受け取る情報をヘッダに詰め込みます。
```c
ElfHandler_t eh;
eh.exec_cmd = strdup(argv[1]);
eh.symbol_name = strdup(argv[2]);
```

## ELFファイルを読み込む
```c
#include <sys/mman.h>
...

// read mode
int fd = open(argv[1], O_RDONLY);
// ファイルサイズの取得のためにstatを使用
struct stat st;
fstat(fd, &st);
// fdの内容をmapする (copy-on-write)
eh.mem = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
```
ELFファイルの中身をメモリにmapするために次の`mmap`を用いています。
```c
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```
`mmap`関数は`fd`の中身を`length`分だけメモリにmapします。
`port`, `flags`にはmappingの際のメモリ保護をどのように行うか、などを指定できます。ここでは、「ページはreadable」を表す`PORT_READ`と、copy-on-writeなmappingを生成する`MAP_PRIVATE`を指定しています。

## ELFファイルの検証
読み込んだELFファイルが正当なものかどうかを検証します。
まず`eh.mem`から各ヘッダを得ます。
```c
eh.ehdr = (Elf64_Ehdr *)eh.mem;
eh.phdr = (Elf64_Phdr *)(eh.mem + eh.ehdr->e_phoff);
eh.shdr = (Elf64_Shdr *)(eh.mem + eh.ehdr->e_shoff);
```
このように、ELFヘッダがもつ他2つのヘッダへのoffset情報を利用してsplitできます。

次に、ELFヘッダの情報の正当性を検証します。具体的なコードは省略しますが、検証する項目は以下のとおりです:
- ELFファイルかどうか
- ELF executableかどうか
- x86_64 executableかどうか
- section headerがあるか

## SymbolのAddressを得る
symbol名からsymbolがメモリ上でどのaddrに存在するのかをlookupする関数を作りましょう。
```c
eh.symbol_addr = lookup_symbol_addr_by_name(&eh, eh.symbol_name);
```
関数は次のような実装になります。
```c
Elf64_Addr lookup_symbol_addr_by_name(ElfHandler_t *eh, const char *target_symname) {
    char *str_tbl;
    Elf64_Sym *sym_tbl;
    Elf64_Shdr *cand_shdr;
    uint32_t link_to_str_tbl;
    char *cand_symname;

    // iterate through the section headers
    for (int i = 0; i < eh->ehdr->e_shnum; i++) {
        if (eh->shdr[i].sh_type != SHT_SYMTAB)
            continue;

        cand_shdr = &eh->shdr[i];
        // get the symbol table
        sym_tbl = (Elf64_Sym *)&eh->mem[cand_shdr->sh_offset];
        // get the linked string table
        link_to_str_tbl = cand_shdr->sh_link;
        str_tbl = (char *)&eh->mem[eh->shdr[link_to_str_tbl].sh_offset];

        // iterate through the symbol table
        for (int j = 0; j < eh->shdr[i].sh_size / sizeof(Elf64_Sym); j++, sym_tbl++) {
            // check if the symbol name matches
            cand_symname = &str_tbl[sym_tbl->st_name];
            if (strcmp(cand_symname, target_symname) == 0) {
                return (sym_tbl->st_value);
            }
        }
    }
    return 0;
}
```
symbol tableは、symbol nameをkey、symbol addressをvalueとするエントリの配列です。
この関数でやっていることは、
1. 複数あるsection headerを**総当り**。symbol tableのtypeなら続ける。
2. **symbol table**をgetする。
3. linkされている**string table**(symbol nameのtable)をgetする。
4. symbol tableのエントリを**総当り**。目的のsymbol nameに一致するentryがあればその値(addr)を返す。

という処理です。これでtraceに必要な情報が揃いました。


## traceeの実行
親プロセス(debugger)から子プロセスを生やし、実行ファイルを実行させます。その際、`ptrace`に`PTRACE_TRACEME`というrequestを指定して呼び出しておきます。これによりtraceが可能になります。
子プロセスの生成には`fork/wait`を使えばよいです。
```c
// process id
int pid = fork();
...
// child executes the given program
if (pid == 0) {
    ptrace(PTRACE_TRACEME, 0, NULL, NULL);
    execve(eh.exec_cmd, args, envp);
    exit(EXIT_SUCCESS);
}
int status;
wait(&status);
```

## breakpointの設置
breakpointを設置するには、`symbol_addr`に対応する命令をtrap命令に書き換えます。
具体的には、x86_64の`INT 3`というソフトウェア割り込みを発生させる命令を使用します。この命令はSIGTRAPシグナルを発生させ、処理を中断させます。これを用いてbreakpointの機能を実現します。

書き換える前の元の命令(`orig`)も必要になるので読んでおきましょう。traceeのdataの読み書きは`PTRACE_PEEKTEXT`, `PTRACE_POKETEXT`で実現できたことを思い出すと、以下のような実装になります。
```c
// trap命令のopcode
#define OPCODE_INT3 0xcc
...

// get original instruction
const long original_inst = ptrace(PTRACE_PEEKTEXT, pid, eh.symbol_addr, NULL);
// modify to trap instruction
const long trap_inst = (original_inst & ~0xff) | OPCODE_INT3;
ptrace(PTRACE_POKETEXT, pid, eh.symbol_addr, trap_inst);
```


## メインループ
メインループでは、以下を繰り返します:
- (止まっていた)プロセスの実行を再開
- `SIGTRAP`signalを検知したら
    - レジスタの情報を取得・表示
    - `trap`を元の命令に書き換える
    - 命令ポインタを1つ戻す
    - レジスタを復元
    - 命令を1つ進める
    - breakpointを再度復元

実装は次のようになります。
```c
while (1) {
    // resume process execution
    ptrace(PTRACE_CONT, pid, NULL, NULL);
    wait(&status);

    if (WIFEXITED(status)) {
        break;
    }
    if (WIFSTOPPED(status) && WSTOPSIG(status) == SIGTRAP) {
        // get registers info and display them
        ptrace(PTRACE_GETREGS, pid, NULL, &eh.regs);
        display_registers(&eh);
        printf("\nPlease hit [ENTER] key to continue: ");
        getchar();
        // restore original instruction
        ptrace(PTRACE_POKETEXT, pid, eh.symbol_addr, original_inst);
        // single step to execute the original instruction
        eh.regs.rip -= 1;
        ptrace(PTRACE_SETREGS, pid, NULL, &eh.regs);
        ptrace(PTRACE_SINGLESTEP, pid, NULL, NULL);
        wait(NULL);
        // restore trap instruction
        ptrace(PTRACE_POKETEXT, pid, eh.symbol_addr, trap_inst);
    }
}
```
ptrace requestの`GETREGS`/`SETREGS`でレジスタのget/setをしている点、命令ポインタの値を1減らして1step前に戻している点、`SINGLESTEP`で1step進めている点がコアと言えるでしょう。


# 結果
完成した自作デバッガで簡単なプログラムをデバッグしてみましょう。

## テストプログラム
```c
// test_add.c
#include <stdio.h>

int add (int a, int b, int c) {
    return a + b + c;
}

int main(int argc, char **argv, char **envp) {
    int a = 1;
    int b = 2;
    int c = 9;
    printf("a = %d, b = %d, c = %d\n", a, b, c);

    int d = add(a, b, 23); // 1(1) + 2(2) + 23(17) = 26(1a)
    printf("%d + %d + %d = %d\n", a, b, 23, d);
    int e = add(d, c, 54); // 26(1a) + 9(9) + 54(36) = 89(59)
    printf("%d + %d + %d = %d\n", d, c, 54, e);
    int f = add(e, 1, 7);  // 89(59) + 1(1) + 7(7) = 97(61)
    printf("%d + %d + %d = %d\n", e, 1, 7, f);
    return 0;
}
```
C言語のソースファイルをELF executableにコンパイルするには、`-no-pie`というoptionを付けて
```bash
$ gcc -no-pie test_add.c -o test_add
```
のようにコンパイルします。

## デバッガを使ってみよう

`test_add`の関数`add`にbreakpointを貼ってみましょう。
```bash
$ ./debugger test_add add
```
`add`関数は計3回呼び出されているため、3回breakpointに引っかかって停止し、その度にレジスタ情報が表示されます。それぞれ見てみましょう。
特に、関数`add`の引数の値と、レジスタ`rdi, rsi, rdx`の値が一致していることを確認します。ここで使われるレジスタはx86の関数呼び出し規約によります。


- 1回目 `int d = add(a, b, 23); // 1(1) + 2(2) + 23(17) = 26(1a)`
```bash
$ ./debugger ./test_add add
Tracing pid:43130 at symbol addr 401136
a = 1, b = 2, c = 9

%rax: 1
%rbx: 401260
%rcx: 2         
%rdx: 17        // add関数の第3引数
%rsi: 2         // add関数の第2引数
%rdi: 1         // add関数の第1引数
%rbp: 7ffd2c1049d0
%rsp: 7ffd2c104988
%r8: 0
%r9: 14
%r10: 40201a
%r11: 246
%r12: 401050
%r13: 7ffd2c104ac0
%r14: 0
%r15: 0
%rip: 401137
%rflags: 206
%cs: 33
%ss: 2b
%ds: 0
%es: 0
%fs: 0
%gs: 0

Please hit [ENTER] key to continue: 
```

- 2回目 `int e = add(d, c, 54); // 26(1a) + 9(9) + 54(36) = 89(59)`

```bash
1 + 2 + 23 = 26

%rax: 1a       // 前回の計算結果?
%rbx: 401260
%rcx: 9        
%rdx: 36       // add関数の第3引数
%rsi: 9        // add関数の第2引数
%rdi: 1a       // add関数の第1引数
...
%gs: 0

Please hit [ENTER] key to continue: 
```

- 3回目 `int f = add(e, 1, 7);  // 89(59) + 1(1) + 7(7) = 97(61)`
```bash
26 + 9 + 54 = 89

%rax: 59       // 前回の計算結果?
%rbx: 401260
%rcx: 0
%rdx: 7        // add関数の第3引数
%rsi: 1        // add関数の第2引数
%rdi: 59       // add関数の第1引数
...
%gs: 0

Please hit [ENTER] key to continue: 
89 + 1 + 7 = 97
Completed tracing pid: 43130
```

いずれの場合も一致していることが確認できましたね！
以上で関数名(`symbol_name`)を指定し、breakpointを貼る機能をもつデバッガを作ることができました。

# おわりに
ごくごく小さな車輪の再発明でしたが、デバッガの仕組みが理解できて非常に楽しかったです。学科のOSの授業で扱った`fork`, `wait`, `exec`などのシステムコールも復習できました。
on goingなプロセスにattachする機能をもつデバッガの実装は今回は時間の都合上諦めましたが、`PTRACE_ATTACH`などのrequestを適切に使えば、そこまで難しい話ではないようです。みなさんも是非デバッガを自作して仕組みの理解を深めてみて下さい！あるいは、よりアドバンスドなデバッガ自作に挑戦してみてください！

## 余談: 車輪の再発明っていいよね
ここからは余談です。
「エンジニアたるもの作るものがなきゃ！」という言説をよく見かけます。ここではこのような言説に対する立場は述べませんが、私が大学に入学してからプログラミングを始めたときは「何かアプリを作りたい」「hogeというサービスを立ち上げた」というモチベーションはあまり無く、どちらかというと「コンピュータおもしろい」「コード書くの楽しい」といったモチベーションでした。
それ以降SWEのアルバイト・インターンを経験して徐々に作りたいもの・携わりたいものが分かってきましたが、個人で応用的な何かを作る、というのは学科の実験を除いてあまりモチベーションが湧いていません。
そんな人間である私にとっては、既に世にある基盤的なソフトウェア(の簡単なもの)を自作するという車輪の再発明はモチベーションを保ちやすく、得るものも非常に大きく感じます。
この世には既に色々な〇〇自作[^2]があります。私と似た雰囲気のある人は是非色々な自作に取り組んでみてください！おすすめです！

## Reference
- はじめてのgdb, https://qiita.com/arene-calix/items/a08363db88f21c81d351
- ELF Formatについて, https://www.hazymoon.jp/OpenBSD/annex/elf.html
- 最小限のELF, https://keens.github.io/blog/2020/04/12/saishougennoelf/
- ptraceシステムコール入門 ― プロセスの出力を覗き見してみよう！, https://itchyny.hatenablog.com/entry/2017/07/31/090000
- Ryan "elfmaster" O'Neill, Learning Linux Binary Analysis, 2016, Packt
- man page of MMAP, https://linuxjm.osdn.jp/html/LDP_man-pages/man2/mmap.2.html
- x86_64で関数の引数とレジスタの対応を確認する (アセンブラ), https://qiita.com/hara0219/items/6556ef17d00922536fa8



[^1]: 大規模ソフトウェアは既に[記事](https://zenn.dev/irugo/articles/06a67373aa713a)になっていて、プロセッサ実験はあまり良いものができず、人工知能演習は優勝したがためにまだ内容を世に出せない
[^2]: 自作CPU、自作OS、自作言語処理系、などなど