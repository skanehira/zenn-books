---
title: "実行バイナリの生成"
---

最終章である。
ここまで実装したパーツを全部つなげ、CLIから叩ける`tiny-linker`コマンドとして仕上げる。
最後には実際に`a.out`を生成して動かしてみる、本書の集大成となる章である。

## 本章で実装するファイル

```
src/
├── main.rs            # 本章
└── linker.rs          # 本章（link_to_file追加）
```

## link_to_file関数の実装

各処理を順番に呼ぶだけのシンプルなオーケストレーション関数を、`Linker`に追加する。
8章で全体像として見せた疑似コードを、ほぼそのまま実装に落とす形になる。

### テストを書く

```diff:src/linker.rs
+use std::io::Cursor;
+
+use crate::error::{LinkerError, Result};
 use crate::parser::Elf;

 #[derive(Debug, Default)]
 pub struct Linker {
     pub objects: Vec<Elf>,
     pub object_names: Vec<String>,
 }

 impl Linker {
     pub fn add_object(&mut self, name: String, obj: Elf) {
         self.object_names.push(name);
         self.objects.push(obj);
     }
+
+    pub fn link_to_file(&mut self, _inputs: Vec<Vec<u8>>) -> Result<Vec<u8>> {
+        todo!()
+    }
 }
```

```diff:src/linker.rs
 #[cfg(test)]
 mod tests {
     use super::*;

+    #[test]
+    fn link_to_file_returns_valid_elf() {
+        let main_o = include_bytes!("parser/fixtures/main.o").to_vec();
+        let sub_o = include_bytes!("parser/fixtures/sub.o").to_vec();
+
+        let mut linker = Linker::default();
+        let output = linker.link_to_file(vec![main_o, sub_o]).unwrap();
+
+        // ELFマジックナンバー
+        assert_eq!(&output[0..4], &[0x7f, b'E', b'L', b'F']);
+
+        // ファイルタイプ（EXEC = 2）
+        assert_eq!(output[16], 2);
+
+        // マシン（AArch64 = 0xb7）
+        assert_eq!(output[18], 0xb7);
+    }
 }
```

### link_to_file関数を実装する

```diff:src/linker.rs
     pub fn link_to_file(&mut self, inputs: Vec<Vec<u8>>) -> Result<Vec<u8>> {
-        todo!()
+        // 1. 入力ファイルをパース
+        for (idx, input) in inputs.iter().enumerate() {
+            let obj = Elf::parse(input).map_err(|e| LinkerError::Parse(format!("{:?}", e)))?;
+            self.add_object(format!("input_{}", idx), obj);
+        }
+
+        // 2. シンボル解決
+        let mut resolved_symbols = self.resolve_symbols()?;
+
+        // 3. セクション配置
+        let (output_sections, section_name_offsets) =
+            self.layout_sections(&mut resolved_symbols)?;
+
+        // 4. 実行ファイルを書き出し
+        let mut out = Cursor::new(Vec::new());
+        self.write_executable(
+            &mut out,
+            resolved_symbols,
+            output_sections,
+            section_name_offsets,
+        )?;
+
+        Ok(out.into_inner())
     }
```

### テストを実行する

```sh
$ cargo test linker::tests::link_to_file
running 1 test
test linker::tests::link_to_file_returns_valid_elf ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## main関数の実装

CLIから叩けるようにする。
出力ファイルには実行権限が必要なので、`OpenOptions`で`mode(0o755)`を指定して開いている。

```rust:src/main.rs
use std::env;
use std::io::Write;
use std::os::unix::fs::OpenOptionsExt;
use std::path::Path;
use std::process;

use tiny_linker::error::LinkerError;
use tiny_linker::linker::Linker;

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
    let mut linker = Linker::default();
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
        .mode(0o755) // rwxr-xr-x
        .open(path)
}
```

## 動作確認

ようやく実際に動かす段階である。
2章で作成した`main.c`と`sub.c`を題材に、自作リンカーで`a.out`を生成してみる。

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
$ ./target/release/tiny-linker a.out main.o sub.o
Linked successfully: a.out
```

### 3. 実行確認

```sh
$ ./a.out
$ echo $?
11
```

終了ステータスが`11`になっていれば成功。

### 4. readelfでの確認

念のため、生成されたELFファイルを`readelf`でも見てみる。

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
  Start of section headers:          472 (bytes into file)
  ...

$ readelf -S a.out
There are 6 section headers, starting at offset 0x1d8:

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
     1: 0000000000410110     0 NOTYPE  LOCAL  DEFAULT    2 $d
     2: 0000000000400100     0 NOTYPE  LOCAL  DEFAULT    1 $x
     3: 0000000000400100     0 NOTYPE  GLOBAL DEFAULT    1 _start
     4: 0000000000410110     4 OBJECT  GLOBAL DEFAULT    2 x
```

エントリポイントもセクション配置もシンボルテーブルも、想定どおりの形になっている。

補足: `$d`と`$x`は`gcc`がARM64向けに自動で付与する mapping symbol で、コードとデータの境界を示すために使われている。

## まとめ

本書ではRustで小さなリンカーを実装した。
次の5ステップで、リンカーが何をやっているのかを一通り理解出来たと思う。

1. ELFパーサー：`nom`を使ってELFバイナリをパース
2. シンボル解決：未定義シンボルと定義済みシンボルを紐付け
3. セクション配置：`.text`と`.data`をマージして、アドレスを割り当て
4. 再配置：`R_AARCH64_ADR_PREL_LO21`の処理を実装
5. ELF出力：実行可能なELFファイルを生成

本書では、簡単なリンカーの実装のみだったが、もう少し深く知りたい方はぜひ、こちらの本を買って読んでみてほしい。
本書の内容ももちろん、より詳しくリンカー周りについて書かれているので、深堀りするには丁度良い本となっている。

https://www.amazon.co.jp/dp/4789838072
