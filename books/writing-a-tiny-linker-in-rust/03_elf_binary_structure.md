---
title: "ELFバイナリの構造"
---

本章では、オブジェクトファイルの概要を説明した後、`readelf`の出力と`/usr/include/elf.h`の構造体定義を照らし合わせながら、ELFバイナリの構造を理解していく。

## オブジェクトファイルとは

オブジェクトファイルとは、コンパイラがソースコード（CやRustなど）をコンパイルした結果の中間ファイルである。本書で扱う`main.o`や`sub.o`がオブジェクトファイルにあたる。

オブジェクトファイルはOSによってフォーマットが異なる。

| OS | フォーマット |
|----|-------------|
| Unix/Linux | ELF（Executable and Linkable Format） |
| macOS | Mach-O（Mach Object） |
| Windows | COFF（Common Object File Format） |

本書で実装するリンカーはARM64 Linux向けなので、ELFフォーマットを扱う。

## ELFファイルの全体構造

ELFファイルは次のような構造になっている。

```
┌──────────────────┐
│ ELF Header       │  ← ファイルの基本情報
├──────────────────┤
│ Program Header   │  ← 実行可能ファイル用（OSがメモリにロードする際に使用）
├──────────────────┤
│ .text            │  ← コード（機械語）
├──────────────────┤
│ .data            │  ← 初期化済みデータ
├──────────────────┤
│ .symtab          │  ← シンボルテーブル
├──────────────────┤
│ .strtab          │  ← 文字列テーブル（シンボル名）
├──────────────────┤
│ .rela.text       │  ← リロケーション情報
├──────────────────┤
│ ...              │
├──────────────────┤
│ Section Header   │  ← リンカー用（各セクションの情報）
│ Table            │
└──────────────────┘
```

オブジェクトファイルはリンカーが読み取るためのファイルなので、セクションヘッダーテーブルが重要となる。一方、実行可能ファイルはOSが実行するためのファイルなので、プログラムヘッダーテーブルが重要となる。

以降、前章で作成した`sub.o`を使って各構造を詳しく説明する。

## readelfコマンド

`readelf`コマンドを使うとELFファイルの情報を確認できる。主なオプションは次のとおり。

| オプション | 説明 |
|-----------|------|
| `-h` | ELFヘッダーを表示 |
| `-S` | セクションヘッダーを表示 |
| `-s` | シンボルテーブルを表示 |
| `-r` | リロケーション情報を表示 |
| `-l` | プログラムヘッダーを表示 |
| `-a` | すべての情報を表示 |

## ELFヘッダー

ELFヘッダーはELFファイルの先頭64バイトに配置され、ファイル全体のメタデータを含む。

### readelfでの確認

```sh
$ readelf -h sub.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  ...
  Type:                              REL (Relocatable file)
  Machine:                           AArch64
  ...
  Start of section headers:          416 (bytes into file)
  ...
  Number of section headers:         9
  Section header string table index: 8
```

本書のリンカー実装で必要な項目を抜粋して説明する。

### 主要フィールドの説明

| readelf出力 | 構造体フィールド | 説明 |
|-------------|-----------------|------|
| Magic | `e_ident[0-3]` | マジックナンバー（`0x7f, 'E', 'L', 'F'`）。ELFファイルであることを識別 |
| Class | `e_ident[4]` | ファイルクラス。2ならELF64 |
| Data | `e_ident[5]` | エンディアン。1ならリトルエンディアン |
| Type | `e_type` | ファイルタイプ。1=REL（オブジェクトファイル）、2=EXEC（実行可能） |
| Machine | `e_machine` | アーキテクチャ。183=AArch64 |
| Start of section headers | `e_shoff` | セクションヘッダーテーブルのファイル内オフセット |
| Number of section headers | `e_shnum` | セクションの数 |
| Section header string table index | `e_shstrndx` | セクション名文字列テーブルのインデックス |

パーサーを実装する際は、`e_shoff`と`e_shnum`を使ってセクションヘッダーの位置と数を特定する。`e_shstrndx`はセクション名を取得する際に使用する。

### e_ident（ELF識別子）

`e_ident`は16バイトの配列で、ELFファイルの基本情報を格納する。名前列は`elf.h`で定義されている定数名である。

| オフセット | 名前                 | 説明             | 値の例                |
|------------|----------------------|------------------|-----------------------|
| 0-3        | `EI_MAG0`〜`EI_MAG3` | マジックナンバー | `0x7f, 'E', 'L', 'F'` |
| 4          | `EI_CLASS`           | ファイルクラス   | 2=64bit               |
| 5          | `EI_DATA`            | エンディアン     | 1=リトルエンディアン  |
| 6          | `EI_VERSION`         | ELFバージョン    | 1                     |
| 7          | `EI_OSABI`           | OS/ABI           | 0=System V            |
| 8-15       | `EI_PAD`             | パディング       | 0                     |

パーサーではまずマジックナンバー（`0x7f, 'E', 'L', 'F'`）を確認し、ELFファイルであることを検証する。

## セクションヘッダー

セクションヘッダーは各セクションのメタデータを格納する。ELFヘッダーの`e_shoff`が指す位置から、`e_shnum`個のセクションヘッダーが並んでいる。

### readelfでの確認

```sh
$ readelf -S sub.o
There are 9 section headers, starting at offset 0x1a0:

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
  [ 6] .symtab           SYMTAB           0000000000000000  00000070
       00000000000000d8  0000000000000018           7     8     8
  [ 7] .strtab           STRTAB           0000000000000000  00000148
       000000000000000c  0000000000000000           0     0     1
  [ 8] .shstrtab         STRTAB           0000000000000000  00000154
       0000000000000045  0000000000000000           0     0     1
```

### elf.hの構造体定義

```c:/usr/include/elf.h
typedef struct
{
  Elf64_Word    sh_name;                /* Section name (string tbl index) */
  Elf64_Word    sh_type;                /* Section type */
  Elf64_Xword   sh_flags;               /* Section flags */
  Elf64_Addr    sh_addr;                /* Section virtual addr at execution */
  Elf64_Off     sh_offset;              /* Section file offset */
  Elf64_Xword   sh_size;                /* Section size in bytes */
  Elf64_Word    sh_link;                /* Link to another section */
  Elf64_Word    sh_info;                /* Additional section information */
  Elf64_Xword   sh_addralign;           /* Section alignment */
  Elf64_Xword   sh_entsize;             /* Entry size if section holds table */
} Elf64_Shdr;
```

### readelf出力との対応

| readelf出力 | 構造体フィールド | サイズ | 説明 |
|-------------|-----------------|--------|------|
| Nr | （インデックス） | - | セクションの順番 |
| Name | `sh_name` | 4バイト | `.shstrtab`内のオフセット |
| Type | `sh_type` | 4バイト | セクションの種類 |
| Address | `sh_addr` | 8バイト | 実行時の仮想アドレス |
| Offset | `sh_offset` | 8バイト | ファイル内のオフセット |
| Size | `sh_size` | 8バイト | セクションのサイズ |
| Flags | `sh_flags` | 8バイト | セクションの属性 |
| Link | `sh_link` | 4バイト | 関連セクションへのリンク |
| Info | `sh_info` | 4バイト | 追加情報 |
| Align | `sh_addralign` | 8バイト | アラインメント（配置境界。4なら4バイト境界に配置） |
| EntSize | `sh_entsize` | 8バイト | エントリサイズ（テーブルの場合） |

`sh_name`は文字列テーブル（`.shstrtab`）内のオフセットであり、実際のセクション名はそこから取得する必要がある。

### セクションタイプ（sh_type）

| 値 | 名前 | 説明 | 該当セクション例 |
|----|------|------|----------------|
| 0 | `SHT_NULL` | 未使用 | インデックス0 |
| 1 | `SHT_PROGBITS` | プログラムデータ | `.text`, `.data` |
| 2 | `SHT_SYMTAB` | シンボルテーブル | `.symtab` |
| 3 | `SHT_STRTAB` | 文字列テーブル | `.strtab`, `.shstrtab` |
| 4 | `SHT_RELA` | 再配置エントリ | `.rela.text` |
| 8 | `SHT_NOBITS` | 未初期化データ | `.bss` |

### セクションフラグ（sh_flags）

| 値 | 名前 | readelf表示 | 説明 |
|----|------|-------------|------|
| 0x1 | `SHF_WRITE` | W | 書き込み可能 |
| 0x2 | `SHF_ALLOC` | A | メモリに配置される |
| 0x4 | `SHF_EXECINSTR` | X | 実行可能 |

`.text`セクションは`AX`（Alloc + eXecute）、`.data`セクションは`WA`（Write + Alloc）となっている。

### sh_linkの使い方

`.symtab`セクションの`Link`が`7`になっているのは、シンボル名の文字列テーブルがインデックス7の`.strtab`であることを示している。パーサー実装時は、この値を使って文字列テーブルを参照する。

## 文字列テーブル

文字列テーブルはNULL終端の文字列を連続して格納する。シンボル名やセクション名は、文字列テーブル内のオフセットとして参照される。

```
オフセット: 0   1   2   3   4   5   6   7   8   9  10
データ:    \0   x  \0   _   s   t   a   r   t  \0  ...
```

オフセット1から始まる文字列は`x`、オフセット3から始まる文字列は`_start`となる。

ELFには2種類の文字列テーブルがある。

| テーブル | 用途 | 参照元 |
|----------|------|--------|
| `.strtab` | シンボル名の格納 | シンボルテーブルの`st_name` |
| `.shstrtab` | セクション名の格納 | セクションヘッダーの`sh_name` |

パーサーでは、オフセットからNULLバイトまでを読み取って文字列を取得する。

## シンボルテーブル

シンボルテーブルは関数や変数の情報を格納する。

### readelfでの確認

```sh
$ readelf -s sub.o
Symbol table '.symtab' contains 9 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS sub.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    2
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     5: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    2 $d
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     8: 0000000000000000     4 OBJECT  GLOBAL DEFAULT    2 x
```

### elf.hの構造体定義

```c:/usr/include/elf.h
typedef struct
{
  Elf64_Word    st_name;                /* Symbol name (string tbl index) */
  unsigned char st_info;                /* Symbol type and binding */
  unsigned char st_other;               /* Symbol visibility */
  Elf64_Section st_shndx;               /* Section index */
  Elf64_Addr    st_value;               /* Symbol value */
  Elf64_Xword   st_size;                /* Symbol size */
} Elf64_Sym;
```

### readelf出力との対応

| readelf出力 | 構造体フィールド | サイズ | 説明 |
|-------------|-----------------|--------|------|
| Num | （インデックス） | - | シンボルの順番（再配置から参照される） |
| Value | `st_value` | 8バイト | シンボルの値（アドレスやオフセット） |
| Size | `st_size` | 8バイト | シンボルのサイズ |
| Type | `st_info`の下位4ビット | - | シンボルの種類 |
| Bind | `st_info`の上位4ビット | - | バインディング |
| Vis | `st_other` | 1バイト | 可視性 |
| Ndx | `st_shndx` | 2バイト | 所属セクションのインデックス |
| Name | `st_name` | 4バイト | `.strtab`内のオフセット |

`st_info`は1バイトに2つの情報が格納されている点に注意。パーサーでは上位4ビットと下位4ビットを分離して解釈する必要がある。

### st_infoのエンコーディング

```c
#define ELF64_ST_BIND(info)          ((info) >> 4)
#define ELF64_ST_TYPE(info)          ((info) & 0xf)
#define ELF64_ST_INFO(bind, type)    (((bind) << 4) + ((type) & 0xf))
```

**バインディング（上位4ビット）**

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `STB_LOCAL` | ファイル内のみ有効 |
| 1 | `STB_GLOBAL` | 他ファイルから参照可能 |
| 2 | `STB_WEAK` | 同名のGLOBALがあれば置き換えられる |

**タイプ（下位4ビット）**

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `STT_NOTYPE` | タイプなし |
| 1 | `STT_OBJECT` | データオブジェクト（変数） |
| 2 | `STT_FUNC` | 関数 |
| 3 | `STT_SECTION` | セクション |
| 4 | `STT_FILE` | ファイル名 |

### st_shndxの特殊な値

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `SHN_UNDEF` | 未定義（外部参照） |
| 0xfff1 | `SHN_ABS` | 絶対値 |

`main.o`のシンボル`x`は`Ndx`が`UND`（0）となっており、外部で定義されていることを示す。リンカーはこのような未定義シンボルを解決する。

## 再配置エントリ

再配置エントリはリンク時に調整が必要な場所を示す。

### readelfでの確認

```sh
$ readelf -r main.o
Relocation section '.rela.text' at offset 0x178 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000000  000900000112 R_AARCH64_ADR_PRE 0000000000000000 x + 0
```

### elf.hの構造体定義

```c:/usr/include/elf.h
typedef struct
{
  Elf64_Addr    r_offset;               /* Address */
  Elf64_Xword   r_info;                 /* Relocation type and symbol index */
  Elf64_Sxword  r_addend;               /* Addend */
} Elf64_Rela;
```

### readelf出力との対応

| readelf出力 | 構造体フィールド | サイズ | 説明 |
|-------------|-----------------|--------|------|
| Offset | `r_offset` | 8バイト | 再配置を適用する位置 |
| Info | `r_info` | 8バイト | タイプとシンボルインデックス |
| Type | `r_info`の下位32ビット | - | 再配置タイプ |
| Sym. Value | （シンボルの値） | - | 参照先シンボルの値 |
| Sym. Name | （シンボル名） | - | `r_info`の上位32ビットでシンボルを参照 |
| Addend | `r_addend` | 8バイト | 加算値 |

### r_infoのエンコーディング

```c
#define ELF64_R_SYM(info)            ((info) >> 32)
#define ELF64_R_TYPE(info)           ((info) & 0xffffffff)
#define ELF64_R_INFO(sym, type)      (((Elf64_Xword)(sym) << 32) + (type))
```

`Info`の値`000900000112`を解析すると：
- 上位32ビット: `0x0009` → シンボルインデックス9（`x`）
- 下位32ビット: `0x00000112` = 274 → `R_AARCH64_ADR_PREL_LO21`

### 本書で扱う再配置タイプ

| 値 | 名前 | 説明 |
|----|------|------|
| 274 | `R_AARCH64_ADR_PREL_LO21` | ADR命令用のPC相対アドレス（21ビット） |

## プログラムヘッダー

プログラムヘッダーは実行時にOSがメモリにロードするセグメントを定義する。オブジェクトファイル（`.o`）には存在せず、実行可能ファイルに含まれる。

### readelfでの確認

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

### elf.hの構造体定義

```c:/usr/include/elf.h
typedef struct
{
  Elf64_Word    p_type;                 /* Segment type */
  Elf64_Word    p_flags;                /* Segment flags */
  Elf64_Off     p_offset;               /* Segment file offset */
  Elf64_Addr    p_vaddr;                /* Segment virtual address */
  Elf64_Addr    p_paddr;                /* Segment physical address */
  Elf64_Xword   p_filesz;               /* Segment size in file */
  Elf64_Xword   p_memsz;                /* Segment size in memory */
  Elf64_Xword   p_align;                /* Segment alignment */
} Elf64_Phdr;
```

### readelf出力との対応

| readelf出力 | 構造体フィールド | サイズ | 説明 |
|-------------|-----------------|--------|------|
| Type | `p_type` | 4バイト | セグメントタイプ |
| Offset | `p_offset` | 8バイト | ファイル内のオフセット |
| VirtAddr | `p_vaddr` | 8バイト | 仮想アドレス |
| PhysAddr | `p_paddr` | 8バイト | 物理アドレス |
| FileSiz | `p_filesz` | 8バイト | ファイル内のサイズ |
| MemSiz | `p_memsz` | 8バイト | メモリ内のサイズ |
| Flags | `p_flags` | 4バイト | セグメントフラグ |
| Align | `p_align` | 8バイト | アラインメント |

### セグメントタイプ（p_type）

| 値 | 名前 | 説明 |
|----|------|------|
| 1 | `PT_LOAD` | メモリにロードするセグメント |

### セグメントフラグ（p_flags）

| 値 | 名前 | readelf表示 | 説明 |
|----|------|-------------|------|
| 0x1 | `PF_X` | E | 実行可能 |
| 0x2 | `PF_W` | W | 書き込み可能 |
| 0x4 | `PF_R` | R | 読み取り可能 |

1つ目のLOADセグメント（`R E`）はコード領域、2つ目のLOADセグメント（`RW`）はデータ領域である。

## まとめ

本章では、readelf出力とelf.h定義を照らし合わせながら、ELFバイナリの構造を学んだ。

パーサー実装において重要なポイント：

1. **ELFヘッダー**から`e_shoff`と`e_shnum`を読み取り、セクションヘッダーの位置を特定する
2. **セクションヘッダー**の`sh_link`を使って、関連する文字列テーブルを参照する
3. **文字列テーブル**からオフセットを使って文字列を取得する
4. **シンボルテーブル**の`st_info`は上位4ビットと下位4ビットに分離して解釈する
5. **再配置エントリ**の`r_info`も同様に上位32ビットと下位32ビットに分離する

次章では、この知識を使ってELFパーサーを実装していく。
