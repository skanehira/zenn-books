---
title: "ELFパーサーの実装（1）ヘッダー"
---

本章からELFパーサーの実装を始める。まずはプロジェクトのセットアップとELFヘッダーのパースを実装する。

## プロジェクトのセットアップ

Rustプロジェクトを作成する。

```sh
$ cargo new tiny-linker
$ cd tiny-linker
```

`Cargo.toml`に依存関係を追加する。

```toml:Cargo.toml
[package]
name = "tiny-linker"
version = "0.1.0"
edition = "2024"

[dependencies]
nom = "8.0"
thiserror = "2.0"

[dev-dependencies]
pretty_assertions = "1.4"
```

[nom](https://github.com/Geal/nom)はRustで書かれた高速なパーサーコンビネーターライブラリで、バイナリデータのパースに適している。パーサーコンビネーターとは、小さなパーサーを組み合わせて複雑なパーサーを構築する手法である。

## ディレクトリ構成

最終的なディレクトリ構成は次のようになる。本章では太字のファイルを実装する。

```
src/
├── main.rs
├── lib.rs           # 本章
├── elf.rs           # 本章
├── elf/
│   ├── header.rs    # 本章
│   ├── section.rs
│   ├── symbol.rs
│   ├── relocation.rs
│   └── program_header.rs
├── parser.rs        # 本章
├── parser/
│   ├── error.rs     # 本章
│   ├── header.rs    # 本章
│   ├── section.rs
│   ├── symbol.rs
│   ├── relocation.rs
│   └── helper.rs
└── linker/
    ├── mod.rs
    ├── symbol.rs
    ├── section.rs
    ├── relocation.rs
    ├── output.rs
    └── writer.rs
```

## モジュール構造を作成する

LSPが正しく動作するように、最初にモジュール構造を作成する。

### lib.rsを作成する

```rust:src/lib.rs
pub mod elf;
pub mod parser;
```

### elf.rsを作成する

```rust:src/elf.rs
pub mod header;
```

### parser.rsを作成する

```rust:src/parser.rs
pub mod error;
pub mod header;

use error::ParseError;

pub type ParseResult<'a, T> = nom::IResult<&'a [u8], T, ParseError>;
```

### 空のモジュールファイルを作成する

LSPが動作するように、空のファイルを作成しておく。

```sh
$ mkdir -p src/elf src/parser
$ touch src/elf/header.rs src/parser/error.rs src/parser/header.rs
```

## テストフィクスチャの準備

テストに使用するオブジェクトファイルを作成する。2章で作成した`sub.c`をコンパイルしてフィクスチャとして配置する。

```sh
$ mkdir -p src/parser/fixtures
$ gcc -c sub.c -o src/parser/fixtures/sub.o
```

## パーサーエラーを定義する

`src/parser/error.rs`を実装する。

```rust:src/parser/error.rs
use thiserror::Error;

#[derive(Debug, PartialEq, Error)]
pub enum ParseError {
    #[error("File type is not ELF: {0:?}")]
    FileTypeNotELF([u8; 4]),

    #[error("Invalid header size: {0}")]
    InvalidHeaderSize(u8),

    #[error("Invalid class: {0}")]
    InvalidClass(u8),

    #[error("Invalid data encoding: {0}")]
    InvalidData(u8),

    #[error("Invalid ident version: {0}")]
    InvalidIdentVersion(u8),

    #[error("Invalid OS/ABI: {0}")]
    InvalidOSABI(u8),

    #[error("Invalid type: {0}")]
    InvalidType(u16),

    #[error("Invalid machine: {0}")]
    InvalidMachine(u16),

    #[error("Invalid version: {0}")]
    InvalidVersion(u32),
}

impl<I> nom::error::ParseError<I> for ParseError {
    fn from_error_kind(_input: I, _kind: nom::error::ErrorKind) -> Self {
        ParseError::InvalidHeaderSize(0)
    }

    fn append(_input: I, _kind: nom::error::ErrorKind, other: Self) -> Self {
        other
    }
}

impl<I> nom::error::FromExternalError<I, ParseError> for ParseError {
    fn from_external_error(_input: I, _kind: nom::error::ErrorKind, e: ParseError) -> Self {
        e
    }
}
```

## ELFヘッダーのデータ構造を定義する

`src/elf/header.rs`を実装する。

```rust:src/elf/header.rs
#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u8)]
pub enum Class {
    #[default]
    None = 0,
    Bit32 = 1,
    Bit64 = 2,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u8)]
pub enum Data {
    #[default]
    None = 0,
    Lsb = 1, // little-endian
    Msb = 2, // big-endian
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u8)]
pub enum IdentVersion {
    #[default]
    None = 0,
    Current = 1,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u8)]
pub enum OSABI {
    #[default]
    SystemV = 0,
}

#[derive(Default, Debug, PartialEq, Eq)]
pub struct Ident {
    pub class: Class,
    pub data: Data,
    pub version: IdentVersion,
    pub os_abi: OSABI,
    pub abi_version: u8,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u16)]
pub enum Type {
    #[default]
    None = 0,
    Rel = 1,
    Exec = 2,
    Dyn = 3,
    Core = 4,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u16)]
pub enum Machine {
    #[default]
    None = 0,
    X86_64 = 62,
    AArch64 = 183,
    RiscV = 243,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u32)]
pub enum Version {
    #[default]
    None = 0,
    Current = 1,
}

#[derive(Default, Debug, PartialEq, Eq)]
pub struct Header {
    pub ident: Ident,
    pub r#type: Type,
    pub machine: Machine,
    pub version: Version,
    pub entry: u64,
    pub phoff: u64,
    pub shoff: u64,
    pub flags: u32,
    pub ehsize: u16,
    pub phentsize: u16,
    pub phnum: u16,
    pub shentsize: u16,
    pub shnum: u16,
    pub shstrndx: u16,
}
```

## ELFヘッダーのパースを実装

### テストを書く

`src/parser/header.rs`にテストを書く。

```rust:src/parser/header.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::elf::header::{Class, Data, Machine, Type};

    #[test]
    fn parse_elf_header() {
        let raw = include_bytes!("./fixtures/sub.o");
        let (_, header) = parse(raw).unwrap();

        assert_eq!(header.ident.class, Class::Bit64);
        assert_eq!(header.ident.data, Data::Lsb);
        assert_eq!(header.r#type, Type::Rel);
        assert_eq!(header.machine, Machine::AArch64);
        assert_eq!(header.shnum, 9);
    }
}
```

### パース処理を実装する

```diff:src/parser/header.rs
+use super::ParseResult;
+use crate::elf::header::{
+    Class, Data, Header, Ident, IdentVersion, Machine, OSABI, Type, Version,
+};
+use crate::parser::error::ParseError;
+use nom::combinator::map_res;
+use nom::multi::count;
+use nom::number::complete::{le_u16, le_u32, le_u64, le_u8};
+use nom::Parser as _;
+
+const ELF_MAGIC_NUMBER: [u8; 4] = [0x7f, 0x45, 0x4c, 0x46];
+const ELF_IDENT_HEADER_SIZE: usize = 16;
+
 #[cfg(test)]
 mod tests {
```

各列挙型にバイト値からの変換を実装する。

```diff:src/parser/header.rs
 const ELF_MAGIC_NUMBER: [u8; 4] = [0x7f, 0x45, 0x4c, 0x46];
 const ELF_IDENT_HEADER_SIZE: usize = 16;

+impl TryFrom<u8> for Class {
+    type Error = ParseError;
+    fn try_from(b: u8) -> Result<Self, Self::Error> {
+        match b {
+            0 => Ok(Self::None),
+            1 => Ok(Self::Bit32),
+            2 => Ok(Self::Bit64),
+            _ => Err(ParseError::InvalidClass(b)),
+        }
+    }
+}
+
+impl TryFrom<u8> for Data {
+    type Error = ParseError;
+    fn try_from(b: u8) -> Result<Self, Self::Error> {
+        match b {
+            0 => Ok(Self::None),
+            1 => Ok(Self::Lsb),
+            2 => Ok(Self::Msb),
+            _ => Err(ParseError::InvalidData(b)),
+        }
+    }
+}
+
+impl TryFrom<u8> for IdentVersion {
+    type Error = ParseError;
+    fn try_from(b: u8) -> Result<Self, Self::Error> {
+        match b {
+            0 => Ok(Self::None),
+            1 => Ok(Self::Current),
+            _ => Err(ParseError::InvalidIdentVersion(b)),
+        }
+    }
+}
+
+impl TryFrom<u8> for OSABI {
+    type Error = ParseError;
+    fn try_from(b: u8) -> Result<Self, Self::Error> {
+        match b {
+            0 => Ok(Self::SystemV),
+            _ => Err(ParseError::InvalidOSABI(b)),
+        }
+    }
+}
+
+impl TryFrom<u16> for Type {
+    type Error = ParseError;
+    fn try_from(b: u16) -> Result<Self, Self::Error> {
+        match b {
+            0 => Ok(Self::None),
+            1 => Ok(Self::Rel),
+            2 => Ok(Self::Exec),
+            3 => Ok(Self::Dyn),
+            4 => Ok(Self::Core),
+            _ => Err(ParseError::InvalidType(b)),
+        }
+    }
+}
+
+impl TryFrom<u16> for Machine {
+    type Error = ParseError;
+    fn try_from(b: u16) -> Result<Self, Self::Error> {
+        match b {
+            0 => Ok(Self::None),
+            62 => Ok(Self::X86_64),
+            183 => Ok(Self::AArch64),
+            243 => Ok(Self::RiscV),
+            _ => Err(ParseError::InvalidMachine(b)),
+        }
+    }
+}
+
+impl TryFrom<u32> for Version {
+    type Error = ParseError;
+    fn try_from(b: u32) -> Result<Self, Self::Error> {
+        match b {
+            0 => Ok(Self::None),
+            1 => Ok(Self::Current),
+            _ => Err(ParseError::InvalidVersion(b)),
+        }
+    }
+}
+
 #[cfg(test)]
 mod tests {
```

マジックナンバーとELF識別子のパース関数を実装する。

```diff:src/parser/header.rs
 }

+fn parse_magic_number(raw: &[u8]) -> ParseResult<'_, ()> {
+    if raw.len() < 4 {
+        return Err(nom::Err::Error(ParseError::InvalidHeaderSize(raw.len() as u8)));
+    }
+    if raw[..4] != ELF_MAGIC_NUMBER {
+        let input: [u8; 4] = raw[..4].try_into().unwrap();
+        return Err(nom::Err::Error(ParseError::FileTypeNotELF(input)));
+    }
+    Ok((&raw[4..], ()))
+}
+
+fn parse_ident(raw: &[u8]) -> ParseResult<'_, Ident> {
+    let (rest, _) = parse_magic_number(raw)?;
+
+    let (rest, class) = map_res(le_u8, Class::try_from).parse(rest)?;
+    let (rest, data) = map_res(le_u8, Data::try_from).parse(rest)?;
+    let (rest, version) = map_res(le_u8, IdentVersion::try_from).parse(rest)?;
+    let (rest, os_abi) = map_res(le_u8, OSABI::try_from).parse(rest)?;
+    let (rest, abi_version) = le_u8(rest)?;
+    let (rest, _) = count(le_u8, 7).parse(rest)?; // パディング
+
+    let ident = Ident {
+        class,
+        data,
+        version,
+        os_abi,
+        abi_version,
+    };
+    Ok((rest, ident))
+}
+
 #[cfg(test)]
 mod tests {
```

ELFヘッダー全体をパースする関数を実装する。

```diff:src/parser/header.rs
     Ok((rest, ident))
 }

+pub fn parse(raw: &[u8]) -> ParseResult<'_, Header> {
+    if raw.len() < ELF_IDENT_HEADER_SIZE {
+        return Err(nom::Err::Error(ParseError::InvalidHeaderSize(raw.len() as u8)));
+    }
+
+    let (rest, ident) = parse_ident(raw)?;
+    let (rest, r#type) = map_res(le_u16, Type::try_from).parse(rest)?;
+    let (rest, machine) = map_res(le_u16, Machine::try_from).parse(rest)?;
+    let (rest, version) = map_res(le_u32, Version::try_from).parse(rest)?;
+    let (rest, entry) = le_u64(rest)?;
+    let (rest, phoff) = le_u64(rest)?;
+    let (rest, shoff) = le_u64(rest)?;
+    let (rest, flags) = le_u32(rest)?;
+    let (rest, ehsize) = le_u16(rest)?;
+    let (rest, phentsize) = le_u16(rest)?;
+    let (rest, phnum) = le_u16(rest)?;
+    let (rest, shentsize) = le_u16(rest)?;
+    let (rest, shnum) = le_u16(rest)?;
+    let (rest, shstrndx) = le_u16(rest)?;
+
+    Ok((
+        rest,
+        Header {
+            ident,
+            r#type,
+            machine,
+            version,
+            entry,
+            phoff,
+            shoff,
+            flags,
+            ehsize,
+            phentsize,
+            phnum,
+            shentsize,
+            shnum,
+            shstrndx,
+        },
+    ))
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test  
   Compiling tiny-linker v0.1.0 (/Users/skanehira/dev/github.com/skanehira/tiny-linker)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.50s
     Running unittests src/lib.rs (target/debug/deps/tiny_linker-5558fbb2f7e5f511)

running 1 test
test parser::header::tests::parse_elf_header ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tiny_linker-bfb6a3022e853684)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests tiny_linker

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```

## 不正なELFファイルのテストを追加

マジックナンバーが不正な場合のテストを追加する。

```diff:src/parser/header.rs
 #[cfg(test)]
 mod tests {
     use super::*;
     use crate::elf::header::{Class, Data, Machine, Type};

+    #[test]
+    fn invalid_magic_number() {
+        let input: &[u8] = &[0u8; 64];
+        let err = parse(input).unwrap_err();
+        assert_eq!(
+            err,
+            nom::Err::Error(ParseError::FileTypeNotELF([0, 0, 0, 0]))
+        );
+    }
+
     #[test]
     fn parse_elf_header() {
```

```sh
$ cargo test
   Compiling tiny-linker v0.1.0 (/Users/skanehira/dev/github.com/skanehira/tiny-linker)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.24s
     Running unittests src/lib.rs (target/debug/deps/tiny_linker-5558fbb2f7e5f511)

running 2 tests
test parser::header::tests::invalid_magic_number ... ok
test parser::header::tests::parse_elf_header ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tiny_linker-bfb6a3022e853684)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests tiny_linker

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```

## まとめ

本章ではELFヘッダーのパースを実装した。

- `nom`パーサーコンビネーターを使ってバイナリをパース
- `TryFrom`トレイトを使ってバイト値から列挙型への変換を実装
- `map_res`コンビネーターでパースと変換を組み合わせ

次章では、セクションヘッダーとシンボルテーブルのパースを実装する。
