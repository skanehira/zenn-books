---
title: "リンカ入門"
---

本章では開発環境を構築し、実際にオブジェクトファイルを作成してリンクを行い、`readelf`で中身を確認していく。

## 環境構築

本書ではDocker Composeを使って開発環境を構築する。
ARM64 Linux向けのリンカーを実装するため、Linuxコンテナ内で開発を行う。

まず、プロジェクトのルートディレクトリに`compose.yaml`を作成する。

```yaml:compose.yaml
services:
  linker-develop:
    image: rust:slim-bullseye
    container_name: linker-develop
    volumes:
      - .:/work
      - cargo-cache:/root/.cargo
    working_dir: /work
    tty: true
    entrypoint: /bin/bash

volumes:
  cargo-cache:
    name: linker-develop-cargo-cache
```

次のコマンドでコンテナを起動し、シェルに入る。

```sh
$ docker compose up -d
$ docker compose exec linker-develop bash
```

コンテナ内でGCCをインストールする。

```sh
$ apt-get update && apt-get install -y gcc
```

これで開発環境の準備が整った。

## サンプルプログラムの作成

本書で作成する最もシンプルなELFは、変数を終了ステータスコードとして使用するプログラムである。

まず、2つのソースファイルを作成する。

```c:main.c
__asm__(
      ".global _start\n"
      "_start:\n"
      "    adr     x0, x\n"
      "    ldr     w0, [x0]\n"
      "    mov     x8, #93\n"
      "    svc     #0\n"
)
```

```c:sub.c
int x = 11;
```

`main.c`はインラインアセンブリで`_start`エントリポイントを定義している。処理の内容は次のとおり。

1. `adr x0, x` - 変数`x`のアドレスを`x0`レジスタにロード
2. `ldr w0, [x0]` - `x0`が指すアドレスから値を読み取り`w0`レジスタにロード
3. `mov x8, #93` - システムコール番号93（exit）を`x8`レジスタにセット
4. `svc #0` - システムコールを実行

`sub.c`は変数`x`を値`11`で初期化している。

## オブジェクトファイルの生成

GCCを使ってオブジェクトファイルを生成する。

```sh
$ gcc -c main.c -o main.o
$ gcc -c sub.c -o sub.o
```

`-c`オプションはコンパイルのみを行い、リンクは行わないことを指定する。

## ldを使ったリンク

まず、標準のリンカー`ld`を使ってリンクを行い、動作を確認する。

```sh
$ ld -o a.out main.o sub.o
$ ./a.out
$ echo $?
11
```

終了ステータスが`11`となり、正しくリンクされていることが確認できた。

## readelfを使った確認

### オブジェクトファイルの確認

`readelf`を使って、`sub.o`のELFヘッダーを確認する。

```sh
$ readelf -h sub.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          408 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         9
  Section header string table index: 8
```

`Type`が`REL (Relocatable file)`となっており、これがオブジェクトファイルであることを示している。
また、`Program headers`が0であることから、プログラムヘッダーは存在しないことがわかる。

次に、セクション一覧を確認する。

```sh
$ readelf -S sub.o
There are 9 section headers, starting at offset 0x198:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000000  0000000000000000  AX       0     0     1
  [ 2] .data             PROGBITS         0000000000000000  00000040
       0000000000000004  0000000000000000  WA       0     0     4
  [ 3] .bss              NOBITS           0000000000000000  00000044
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .comment          PROGBITS         0000000000000000  00000044
       0000000000000028  0000000000000001  MS       0     0     1
  [ 5] .note.GNU-stack   PROGBITS         0000000000000000  0000006c
       0000000000000000  0000000000000000           0     0     1
  [ 6] .symtab           SYMTAB           0000000000000000  00000070
       00000000000000d8  0000000000000018           7     8     8
  [ 7] .strtab           STRTAB           0000000000000000  00000148
       0000000000000006  0000000000000000           0     0     1
  [ 8] .shstrtab         STRTAB           0000000000000000  0000014e
       0000000000000045  0000000000000000           0     0     1
```

`.data`セクションに4バイトのデータがあることがわかる。これは変数`x`（int型、4バイト）である。

シンボルテーブルを確認する。

```sh
$ readelf -s sub.o
Symbol table '.symtab' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    2
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     5: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    2 $d
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     8: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    2 x
```

8番目のエントリに変数`x`がグローバルシンボルとして登録されている。`Ndx`が2なので、`.data`セクション（インデックス2）に配置されていることがわかる。

### main.oの確認

`main.o`のシンボルテーブルを確認する。

```sh
$ readelf -s main.o
Symbol table '.symtab' contains 10 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    1 $x
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     8: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT    1 _start
     9: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND x
```

重要なポイントは次のとおり。

- 8番目: `_start`がグローバルシンボルとして定義されている（`Ndx`が1で`.text`セクションにある）
- 9番目: `x`が`UND`（未定義）として参照されている

`main.o`は`x`を参照しているが、その定義は持っていない。リンカーの役割は、この未定義シンボル`x`を`sub.o`の定義と紐付けることである。

リロケーション情報を確認する。

```sh
$ readelf -r main.o
Relocation section '.rela.text' at offset 0x178 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000000  000900000112 R_AARCH64_ADR_PRE 0000000000000000 x + 0
```

`.text`セクションのオフセット0に、シンボル`x`への参照があり、`R_AARCH64_ADR_PREL_LO21`タイプのリロケーションが必要であることがわかる。

### 実行可能ファイルの確認

リンク後の実行可能ファイル`a.out`を確認する。

```sh
$ readelf -h a.out
ELF Header:
  ...
  Type:                              EXEC (Executable file)
  ...
  Entry point address:               0x400078
  ...
```

`Type`が`EXEC (Executable file)`となり、エントリポイントアドレスが設定されていることがわかる。

プログラムヘッダーを確認する。

```sh
$ readelf -l a.out
Elf file type is EXEC (Executable file)
Entry point 0x400078
There are 2 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000088 0x0000000000000088  R E    0x10000
  LOAD           0x0000000000000088 0x0000000000410088 0x0000000000410088
                 0x0000000000000004 0x0000000000000004  RW     0x10000
```

2つのLOADセグメントがあり、1つ目は実行可能（R E）、2つ目は読み書き可能（RW）となっている。

これで、オブジェクトファイルから実行可能ファイルへの変換の流れと、各種情報の確認方法を理解できた。
次章では、ELFバイナリの構造をより詳しく解説していく。
