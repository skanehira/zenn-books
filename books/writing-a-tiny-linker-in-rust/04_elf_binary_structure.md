---
title: "ELFバイナリの構造"
---

本章では、ELFバイナリの構造を詳しく解説する。
リンカーを実装するにあたり、ELFの各構造体のフィールドを理解することが重要となる。

ELFの構造に関しては、Linuxのヘッダーファイル`/usr/include/elf.h`を参照すると、バイナリのレイアウトがそのまま表現されているので参考になる。

## ELFヘッダー

ELFヘッダーはELFファイルの先頭64バイトに配置され、ファイル全体のメタデータを含む。

```c:/usr/include/elf.h
typedef struct
{
  unsigned char e_ident[EI_NIDENT];     /* Magic number and other info */
  Elf64_Half    e_type;                 /* Object file type */
  Elf64_Half    e_machine;              /* Architecture */
  Elf64_Word    e_version;              /* Object file version */
  Elf64_Addr    e_entry;                /* Entry point virtual address */
  Elf64_Off     e_phoff;                /* Program header table file offset */
  Elf64_Off     e_shoff;                /* Section header table file offset */
  Elf64_Word    e_flags;                /* Processor-specific flags */
  Elf64_Half    e_ehsize;               /* ELF header size in bytes */
  Elf64_Half    e_phentsize;            /* Program header table entry size */
  Elf64_Half    e_phnum;                /* Program header table entry count */
  Elf64_Half    e_shentsize;            /* Section header table entry size */
  Elf64_Half    e_shnum;                /* Section header table entry count */
  Elf64_Half    e_shstrndx;             /* Section header string table index */
} Elf64_Ehdr;
```

各フィールドの説明は次のとおり。

| フィールド | サイズ | 説明 |
|-----------|--------|------|
| `e_ident` | 16バイト | マジックナンバーとファイル情報 |
| `e_type` | 2バイト | ファイルタイプ（REL=1, EXEC=2など） |
| `e_machine` | 2バイト | アーキテクチャ（AArch64=183） |
| `e_version` | 4バイト | ELFバージョン |
| `e_entry` | 8バイト | エントリポイントのアドレス |
| `e_phoff` | 8バイト | プログラムヘッダーテーブルのオフセット |
| `e_shoff` | 8バイト | セクションヘッダーテーブルのオフセット |
| `e_flags` | 4バイト | プロセッサ固有のフラグ |
| `e_ehsize` | 2バイト | ELFヘッダーのサイズ（64バイト） |
| `e_phentsize` | 2バイト | プログラムヘッダーエントリのサイズ |
| `e_phnum` | 2バイト | プログラムヘッダーエントリの数 |
| `e_shentsize` | 2バイト | セクションヘッダーエントリのサイズ |
| `e_shnum` | 2バイト | セクションヘッダーエントリの数 |
| `e_shstrndx` | 2バイト | セクション名文字列テーブルのインデックス |

### e_ident（ELF識別子）

`e_ident`は16バイトの配列で、次の情報を含む。

| オフセット | 名前 | 説明 |
|-----------|------|------|
| 0-3 | `EI_MAG0-3` | マジックナンバー（0x7f, 'E', 'L', 'F'） |
| 4 | `EI_CLASS` | ファイルクラス（1=32bit, 2=64bit） |
| 5 | `EI_DATA` | データエンコーディング（1=リトルエンディアン, 2=ビッグエンディアン） |
| 6 | `EI_VERSION` | ELFバージョン |
| 7 | `EI_OSABI` | OS/ABI識別子 |
| 8-15 | `EI_PAD` | パディング（未使用） |

## セクションヘッダー

セクションヘッダーは各セクションのメタデータを格納する。

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

| フィールド | サイズ | 説明 |
|-----------|--------|------|
| `sh_name` | 4バイト | セクション名（.shstrtab内のオフセット） |
| `sh_type` | 4バイト | セクションタイプ |
| `sh_flags` | 8バイト | セクションフラグ |
| `sh_addr` | 8バイト | 実行時の仮想アドレス |
| `sh_offset` | 8バイト | ファイル内のオフセット |
| `sh_size` | 8バイト | セクションサイズ |
| `sh_link` | 4バイト | 関連セクションへのリンク |
| `sh_info` | 4バイト | 追加情報 |
| `sh_addralign` | 8バイト | アラインメント |
| `sh_entsize` | 8バイト | エントリサイズ（テーブルの場合） |

### セクションタイプ

主なセクションタイプは次のとおり。

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `SHT_NULL` | 未使用 |
| 1 | `SHT_PROGBITS` | プログラムデータ（.text, .dataなど） |
| 2 | `SHT_SYMTAB` | シンボルテーブル |
| 3 | `SHT_STRTAB` | 文字列テーブル |
| 4 | `SHT_RELA` | 再配置エントリ（addend付き） |
| 8 | `SHT_NOBITS` | 未初期化データ（.bss） |

### セクションフラグ

主なセクションフラグは次のとおり。

| 値 | 名前 | 説明 |
|----|------|------|
| 0x1 | `SHF_WRITE` | 書き込み可能 |
| 0x2 | `SHF_ALLOC` | メモリに配置される |
| 0x4 | `SHF_EXECINSTR` | 実行可能 |

## シンボルテーブル

シンボルテーブルは関数や変数の情報を格納する。

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

| フィールド | サイズ | 説明 |
|-----------|--------|------|
| `st_name` | 4バイト | シンボル名（.strtab内のオフセット） |
| `st_info` | 1バイト | シンボルタイプとバインディング |
| `st_other` | 1バイト | シンボルの可視性 |
| `st_shndx` | 2バイト | 所属セクションのインデックス |
| `st_value` | 8バイト | シンボルの値（アドレス） |
| `st_size` | 8バイト | シンボルのサイズ |

### st_info

`st_info`は上位4ビットがバインディング、下位4ビットがタイプを表す。

**バインディング**

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `STB_LOCAL` | ローカルシンボル（ファイル内のみ） |
| 1 | `STB_GLOBAL` | グローバルシンボル |
| 2 | `STB_WEAK` | ウィークシンボル |

**タイプ**

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `STT_NOTYPE` | タイプなし |
| 1 | `STT_OBJECT` | データオブジェクト（変数） |
| 2 | `STT_FUNC` | 関数 |
| 3 | `STT_SECTION` | セクション |
| 4 | `STT_FILE` | ファイル名 |

### st_shndx

シンボルが所属するセクションのインデックス。特殊な値として次がある。

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `SHN_UNDEF` | 未定義（外部参照） |
| 0xfff1 | `SHN_ABS` | 絶対値 |

## 再配置エントリ

再配置エントリはリンク時に調整が必要な場所を示す。

```c:/usr/include/elf.h
typedef struct
{
  Elf64_Addr    r_offset;               /* Address */
  Elf64_Xword   r_info;                 /* Relocation type and symbol index */
  Elf64_Sxword  r_addend;               /* Addend */
} Elf64_Rela;
```

| フィールド | サイズ | 説明 |
|-----------|--------|------|
| `r_offset` | 8バイト | 再配置を適用する位置 |
| `r_info` | 8バイト | 再配置タイプとシンボルインデックス |
| `r_addend` | 8バイト | 加算値 |

### r_info

`r_info`は上位32ビットがシンボルインデックス、下位32ビットが再配置タイプを表す。

本書で扱う再配置タイプは次のとおり。

| 値 | 名前 | 説明 |
|----|------|------|
| 274 | `R_AARCH64_ADR_PREL_LO21` | ADR命令用のPC相対アドレス（21ビット） |

## プログラムヘッダー

プログラムヘッダーは実行時にOSがメモリにロードするセグメントを定義する。

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

| フィールド | サイズ | 説明 |
|-----------|--------|------|
| `p_type` | 4バイト | セグメントタイプ |
| `p_flags` | 4バイト | セグメントフラグ |
| `p_offset` | 8バイト | ファイル内のオフセット |
| `p_vaddr` | 8バイト | 仮想アドレス |
| `p_paddr` | 8バイト | 物理アドレス |
| `p_filesz` | 8バイト | ファイル内のサイズ |
| `p_memsz` | 8バイト | メモリ内のサイズ |
| `p_align` | 8バイト | アラインメント |

### セグメントタイプ

| 値 | 名前 | 説明 |
|----|------|------|
| 1 | `PT_LOAD` | ロード可能セグメント |

### セグメントフラグ

| 値 | 名前 | 説明 |
|----|------|------|
| 0x1 | `PF_X` | 実行可能 |
| 0x2 | `PF_W` | 書き込み可能 |
| 0x4 | `PF_R` | 読み取り可能 |

## 文字列テーブル

文字列テーブルはNULL終端の文字列を連続して格納する。
シンボル名やセクション名は、文字列テーブル内のオフセットとして参照される。

```
オフセット: 0   1   2   3   4   5   6   7   8   9  10  11
データ:    \0   x  \0   _   s   t   a   r   t  \0  ...
```

オフセット1から始まる文字列は`x`、オフセット3から始まる文字列は`_start`となる。

これでELFバイナリの構造の理解が深まった。次章からは、この知識を使ってELFパーサーを実装していく。
