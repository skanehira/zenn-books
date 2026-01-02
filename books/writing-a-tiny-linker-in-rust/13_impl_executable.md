---
title: "実行バイナリの生成"
---

本章では、これまで実装してきたすべての処理を統合し、実行可能バイナリを生成する。

## main関数の実装

CLIインターフェースを実装する。

```rust:src/main.rs
use std::env;
use std::io::Write;
use std::os::unix::fs::OpenOptionsExt;
use std::path::Path;
use std::process;

use yui::error::LinkerError;
use yui::linker::Linker;

fn main() -> Result<(), LinkerError> {
    let args: Vec<String> = env::args().collect();

    if args.len() < 3 {
        eprintln!("Usage: {} <output> <input1> [<input2> ...]", args[0]);
        process::exit(1);
    }

    // 入力ファイルを読み込み
    let input_paths: Vec<&Path> = args[2..].iter().map(Path::new).collect();
    let inputs: Vec<Vec<u8>> = input_paths
        .iter()
        .map(|path| std::fs::read(path))
        .collect::<Result<Vec<_>, _>>()?;

    // リンカーを実行
    let mut linker = Linker::new();
    let output = linker.link_to_file(inputs)?;

    // 出力ファイルを作成（実行権限付き）
    let mut out = create_output_file(Path::new(&args[1]))?;
    out.write_all(&output)?;

    println!("Linked successfully: {}", args[1]);

    Ok(())
}

fn create_output_file(path: &Path) -> Result<std::fs::File, std::io::Error> {
    std::fs::OpenOptions::new()
        .write(true)
        .truncate(true)
        .create(true)
        .mode(0o755)  // rwxr-xr-x
        .open(path)
}
```

## Linkerのlink_to_file関数

リンク処理全体を実行する関数を実装する。

```rust:src/linker/mod.rs
use std::io::Cursor;
use crate::error::Result;
use crate::parser;

impl Linker {
    pub fn link_to_file(&mut self, inputs: Vec<Vec<u8>>) -> Result<Vec<u8>> {
        // 1. 入力ファイルをパース
        for (idx, input) in inputs.iter().enumerate() {
            let obj = parser::parse_elf(input)
                .map_err(|e| LinkerError::Parse(format!("{:?}", e)))?
                .1;
            self.add_object(format!("input_{}", idx), obj);
        }

        // 2. シンボル解決
        let mut resolved_symbols = self.resolve_symbols()?;

        // 3. セクション配置
        let (output_sections, section_name_offsets) =
            self.layout_sections(&mut resolved_symbols)?;

        // 4. 実行可能ファイルを書き出し
        let mut out = Cursor::new(Vec::new());
        self.write_executable(
            &mut out,
            resolved_symbols,
            output_sections,
            section_name_offsets,
        )?;

        Ok(out.into_inner())
    }
}
```

## 動作確認

実装したリンカーを使って、実行可能バイナリを生成してみる。

### 1. オブジェクトファイルの作成

```sh
# コンテナ内で実行
$ cat > main.c << 'EOF'
__asm__(
      ".global _start\n"
      "_start:\n"
      "    adr     x0, x\n"
      "    ldr     w0, [x0]\n"
      "    mov     x8, #93\n"
      "    svc     #0\n"
)
EOF

$ cat > sub.c << 'EOF'
int x = 11;
EOF

$ gcc -c main.c -o main.o
$ gcc -c sub.c -o sub.o
```

### 2. リンカーのビルドと実行

```sh
$ cargo build --release
$ ./target/release/yui a.out main.o sub.o
Linked successfully: a.out
```

### 3. 実行確認

```sh
$ ./a.out
$ echo $?
11
```

終了ステータスが`11`となり、正しくリンクされていることが確認できた。

### 4. readelfでの確認

生成されたELFファイルを確認する。

```sh
$ readelf -h a.out
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0x400100
  Start of program headers:          64 (bytes into file)
  Start of section headers:          480 (bytes into file)
  ...

$ readelf -S a.out
There are 6 section headers, starting at offset 0x1e0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000400100  00000100
       0000000000000010  0000000000000000  AX       0     0     4
  [ 2] .data             PROGBITS         0000000000410110  00000110
       0000000000000004  0000000000000000  WA       0     0     4
  ...

$ readelf -s a.out
Symbol table '.symtab' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000400100     0 NOTYPE  GLOBAL DEFAULT    1 _start
     2: 0000000000410110     4 OBJECT  GLOBAL DEFAULT    2 x
     ...
```

## デバッグとトラブルシューティング

### よくある問題

**1. シンボルが解決されない**

```
Error: Unresolved symbols: ["x"]
```

原因: シンボル`x`を定義しているオブジェクトファイルがリンク対象に含まれていない。
対策: 必要なオブジェクトファイルをすべて指定する。

**2. エントリポイントが見つからない**

```
Error: Missing entry point: _start
```

原因: `_start`シンボルが定義されていない。
対策: `_start`を定義したオブジェクトファイルをリンク対象に含める。

**3. 実行時にSegmentation fault**

原因: 再配置が正しく行われていない可能性がある。
対策: `readelf -r`で再配置情報を確認し、デバッグする。

### デバッグのコツ

1. `readelf`を使って各段階の出力を確認
2. 標準の`ld`でリンクした結果と比較
3. `objdump -d`で命令のエンコーディングを確認

## まとめ

本書では、Rustで小さなリンカーを実装した。主なポイントは次のとおり。

1. **ELFパーサー**: `nom`を使ってELFバイナリをパース
2. **シンボル解決**: 未定義シンボルと定義済みシンボルを紐付け
3. **セクション配置**: `.text`と`.data`をマージし、アドレスを割り当て
4. **再配置**: `R_AARCH64_ADR_PREL_LO21`の処理を実装
5. **ELF出力**: 実行可能なELFファイルを生成

## 今後の展望

本書で実装したリンカーは最小限の機能のみをサポートしている。発展として次のような機能を実装できる。

- **動的リンク**: 共有ライブラリのサポート（`.so`ファイル、PLT/GOT）
- **追加の再配置タイプ**: `R_AARCH64_CALL26`、`R_AARCH64_ABS64`など
- **他のアーキテクチャ**: x86_64、RISC-Vなど
- **追加セクション**: `.bss`、`.rodata`など
- **デバッグ情報**: DWARF情報のサポート

リンカーの実装を通じて、コンパイルから実行可能ファイル生成までの流れを深く理解できた。
この知識は、低レベルプログラミングやシステムプログラミングにおいて非常に役立つ。
