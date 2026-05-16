---
title: "ELFパーサーの実装（3）再配置情報"
---

本章では再配置情報のパースを実装し、ELFパーサーを完成させる。
最後に、ヘッダー・セクション・シンボル・再配置の4つを束ねる`Elf`構造体も用意して、外から`Elf::parse`一発で呼び出せるようにする。

## 本章で実装するファイル

```
src/
├── elf/
│   └── relocation.rs    # 本章
├── parser/
│   ├── error.rs         # 本章（エラー追加）
│   └── relocation.rs    # 本章
├── parser.rs            # 本章（ELF構造体追加）
└── ...
```

## モジュール構造を更新する

### elf.rsを更新する

```diff:src/elf.rs
 pub mod header;
+pub mod relocation;
 pub mod section;
 pub mod symbol;
```

### parser.rsを更新する

```diff:src/parser.rs
 pub mod error;
 pub mod header;
 pub mod helper;
+pub mod relocation;
 pub mod section;
 pub mod symbol;

 use error::ParseError;

 pub type ParseResult<'a, T> = nom::IResult<&'a [u8], T, ParseError>;
```

### 空のモジュールファイルを作成する

```sh
$ touch src/elf/relocation.rs src/parser/relocation.rs
```

## テスト用フィクスチャの追加

再配置情報のテストには再配置を持つファイルが必要なので、`main.o`をフィクスチャに追加しておく。

```sh
$ gcc -c main.c -o src/parser/fixtures/main.o
```

## 再配置情報のパース

### データ構造を定義する

`src/elf/relocation.rs`を実装する。
`r_info`は1つの値の中にシンボルインデックスと再配置タイプが詰まっているので、こちらもバラして`Info`構造体に持たせる。

```rust:src/elf/relocation.rs
#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u32)]
pub enum RelocationType {
    #[default]
    Unknown = 0,
    /// ADR命令用のPC相対アドレス（21ビット）
    Aarch64AdrPrelLo21 = 274,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
pub struct Info {
    pub r#type: RelocationType,
    pub symbol_index: u32,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
pub struct Rela {
    pub offset: u64,
    pub info: Info,
    pub addend: i64,
}
```

### パーサーエラーを追加する

```diff:src/parser/error.rs
     #[error("Invalid visibility: {0}")]
     InvalidVisibility(u8),
+
+    #[error("Invalid relocation type: {0}")]
+    InvalidRelocationType(u32),
 }
```

### テストを書く

`src/parser/relocation.rs`にテストを書く。
`main.o`には変数`x`を参照する再配置エントリが1つあるはずなので、それが取れることを確認する。

```rust:src/parser/relocation.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::elf::relocation::RelocationType;

    #[test]
    fn parse_relocations() {
        let raw = include_bytes!("./fixtures/main.o");
        let (_, elf_header) = crate::parser::header::parse(raw).unwrap();
        let (_, section_headers) = crate::parser::section::parse_headers(
            raw,
            elf_header.shoff as usize,
            elf_header.shstrndx as usize,
            elf_header.shnum as usize,
        )
        .unwrap();

        let (_, relocations) = parse(&section_headers).unwrap();

        assert_eq!(relocations.len(), 1);
        assert_eq!(relocations[0].offset, 0);
        assert_eq!(
            relocations[0].info.r#type,
            RelocationType::Aarch64AdrPrelLo21
        );
    }
}
```

### パース処理を実装する

`.rela.text`セクションを探して、`size / entsize`の数だけ再配置エントリを読み取る。
`r_info`は下位32ビットが再配置タイプ、上位32ビットがシンボルインデックスである。

```diff:src/parser/relocation.rs
+use super::ParseResult;
+use crate::elf::relocation::{Info, Rela, RelocationType};
+use crate::elf::section::{Header, SectionType};
+use crate::parser::error::ParseError;
+use nom::combinator::map_res;
+use nom::multi::count;
+use nom::number::complete::{le_i64, le_u64};
+use nom::Parser as _;
+
+impl TryFrom<u32> for RelocationType {
+    type Error = ParseError;
+    fn try_from(value: u32) -> Result<Self, Self::Error> {
+        match value {
+            274 => Ok(Self::Aarch64AdrPrelLo21),
+            _ => Err(ParseError::InvalidRelocationType(value)),
+        }
+    }
+}
+
+impl TryFrom<u64> for Info {
+    type Error = ParseError;
+    fn try_from(value: u64) -> Result<Self, Self::Error> {
+        // 下位32ビットが再配置タイプ
+        let r#type = RelocationType::try_from((value & 0xffffffff) as u32)?;
+        // 上位32ビットがシンボルインデックス
+        let symbol_index = (value >> 32) as u32;
+        Ok(Info {
+            r#type,
+            symbol_index,
+        })
+    }
+}
+
+pub fn parse(section_headers: &[Header]) -> ParseResult<'_, Vec<Rela>> {
+    // .rela.textセクションを探す
+    let Some(rela_section) = section_headers
+        .iter()
+        .find(|h| h.r#type == SectionType::Rela)
+    else {
+        return Ok((&[], vec![]));
+    };
+
+    let entry_count = (rela_section.size / rela_section.entsize) as usize;
+
+    let (rest, relocations) = count(
+        |input| {
+            let (rest, offset) = le_u64(input)?;
+            let (rest, info) = map_res(le_u64, Info::try_from).parse(rest)?;
+            let (rest, addend) = le_i64(rest)?;
+
+            let rela = Rela {
+                offset,
+                info,
+                addend,
+            };
+            Ok((rest, rela))
+        },
+        entry_count,
+    )
+   .parse(rela_section.data.as_slice())?;
+
+    Ok((rest, relocations))
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test parser::relocation
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running unittests src/lib.rs (target/debug/deps/tiny_linker-5558fbb2f7e5f511)

running 1 test
test parser::relocation::tests::parse_relocations ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 4 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tiny_linker-bfb6a3022e853684)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## ELFパーサーの統合

ここまでで部品はそろったので、最後にELF全体を表す`Elf`構造体を用意して、各パーサーを順番に呼ぶだけの`parse`関数を実装する。

### テストを書く

`main.o`を入れて、ヘッダー・シンボル・再配置がまとめて取れることを確認する。

```diff:src/parser.rs
 pub type ParseResult<'a, T> = nom::IResult<&'a [u8], T, ParseError>;
+
+#[cfg(test)]
+mod tests {
+    use super::*;
+
+    #[test]
+    fn parse_elf() {
+        let raw = include_bytes!("./parser/fixtures/main.o");
+        let elf = Elf::parse(raw).unwrap();
+
+        // ヘッダーの確認
+        assert_eq!(elf.header.r#type, crate::elf::header::Type::Rel);
+        assert_eq!(elf.header.machine, crate::elf::header::Machine::AArch64);
+
+        // シンボルの確認
+        assert!(elf.symbols.iter().any(|s| s.name == "_start"));
+        assert!(elf.symbols.iter().any(|s| s.name == "x"));
+
+        // 再配置の確認
+        assert_eq!(elf.relocations.len(), 1);
+    }
+}
```

### ELF構造体を実装する

各パーサーを順番に呼んで結果を1つの構造体に詰めるだけ。
`nom::Err`から`ParseError`への変換は毎回同じパターンになるので、思い切ってシンプルに書いている。

```diff:src/parser.rs
 use error::ParseError;

 pub type ParseResult<'a, T> = nom::IResult<&'a [u8], T, ParseError>;
+
+use crate::elf::header::Header;
+use crate::elf::relocation::Rela;
+use crate::elf::section::Header as SectionHeader;
+use crate::elf::symbol::Symbol;
+
+#[derive(Debug)]
+pub struct Elf {
+    pub header: Header,
+    pub section_headers: Vec<SectionHeader>,
+    pub symbols: Vec<Symbol>,
+    pub relocations: Vec<Rela>,
+}
+
+impl Elf {
+    pub fn parse(raw: &[u8]) -> Result<Self, ParseError> {
+        let (_, header) = header::parse(raw).map_err(|e| match e {
+            nom::Err::Error(e) | nom::Err::Failure(e) => e,
+            _ => ParseError::InvalidHeaderSize(0),
+        })?;
+
+        let (_, section_headers) = section::parse_headers(
+            raw,
+            header.shoff as usize,
+            header.shstrndx as usize,
+            header.shnum as usize,
+        )
+        .map_err(|e| match e {
+            nom::Err::Error(e) | nom::Err::Failure(e) => e,
+            _ => ParseError::InvalidHeaderSize(0),
+        })?;
+
+        let (_, symbols) = symbol::parse(raw, &section_headers).map_err(|e| match e {
+            nom::Err::Error(e) | nom::Err::Failure(e) => e,
+            _ => ParseError::InvalidHeaderSize(0),
+        })?;
+
+        let (_, relocations) = relocation::parse(&section_headers).map_err(|e| match e {
+            nom::Err::Error(e) | nom::Err::Failure(e) => e,
+            _ => ParseError::InvalidHeaderSize(0),
+        })?;
+
+        Ok(Self {
+            header,
+            section_headers,
+            symbols,
+            relocations,
+        })
+    }
+}

 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test parser::tests::parse_elf
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.05s
     Running unittests src/lib.rs (target/debug/deps/tiny_linker-5558fbb2f7e5f511)

running 1 test
test parser::tests::parse_elf ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 5 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tiny_linker-bfb6a3022e853684)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## まとめ

本章で再配置情報のパースを実装し、ELFパーサーが完成した。
これでオブジェクトファイルを読み込んで構造体として扱える状態になった。
ここからがリンカー本体の実装である。次章では、リンク処理が全体としてどんな流れで動くのかをざっくり整理する。
