---
title: "ELFパーサーの実装（1）ヘッダー"
---

本章からELFパーサーの実装を始める。まずはプロジェクトのセットアップとELFヘッダーのパースを実装する。

## nomパーサーコンビネーター

本書では[nom](https://github.com/Geal/nom)というパーサーコンビネーターライブラリを使ってELFバイナリをパースする。
nomはRustで書かれた高速なパーサーコンビネーターで、バイナリデータのパースに適している。

パーサーコンビネーターとは、小さなパーサーを組み合わせて複雑なパーサーを構築する手法である。

## プロジェクトのセットアップ

まず、Rustプロジェクトを作成する。

```sh
$ cargo new yui
$ cd yui
```

`Cargo.toml`に依存関係を追加する。

```toml:Cargo.toml
[package]
name = "yui"
edition = "2021"

[dependencies]
nom = "8.0"
thiserror = "2.0"

[dev-dependencies]
pretty_assertions = "1.4"
```

## ディレクトリ構成

プロジェクトのディレクトリ構成は次のようにする。

```
src/
├── main.rs
├── lib.rs
├── error.rs
├── parser.rs
├── parser/
│   ├── mod.rs
│   ├── header.rs
│   ├── section.rs
│   ├── symbol.rs
│   ├── relocation.rs
│   ├── error.rs
│   └── helper.rs
├── elf.rs
├── elf/
│   ├── mod.rs
│   ├── header.rs
│   ├── section.rs
│   ├── symbol.rs
│   ├── relocation.rs
│   └── program_header.rs
└── linker/
    ├── mod.rs
    ├── symbol.rs
    ├── section.rs
    ├── relocation.rs
    ├── output.rs
    └── writer.rs
```

## ELF構造体の設計

まず、パース結果を格納するELF構造体を定義する。

```rust:src/elf.rs
pub mod header;
pub mod program_header;
pub mod relocation;
pub mod section;
pub mod symbol;
```

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
    // ... 他のOS/ABI
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
    Rel = 1,   // Relocatable file
    Exec = 2,  // Executable file
    Dyn = 3,   // Shared object file
    Core = 4,  // Core file
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

## パーサーエラーの定義

パース時のエラーを定義する。

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
```

## ELFヘッダーのパース

ELFヘッダーをパースする関数を実装する。

```rust:src/parser/header.rs
use super::ParseResult;
use crate::elf::header::{
    Class, Data, Header, Ident, IdentVersion, Machine, OSABI, Type, Version,
};
use crate::parser::error::ParseError;
use nom::combinator::map_res;
use nom::multi::count;
use nom::number::complete::{le_u8, le_u16, le_u32, le_u64};
use nom::Parser as _;

const ELF_MAGIC_NUMBER: [u8; 4] = [0x7f, 0x45, 0x4c, 0x46]; // 0x7f 'E' 'L' 'F'
const ELF_IDENT_HEADER_SIZE: usize = 16;

// TryFrom実装でバイトから列挙型への変換を定義
impl TryFrom<u8> for Class {
    type Error = ParseError;
    fn try_from(b: u8) -> Result<Self, Self::Error> {
        match b {
            0 => Ok(Self::None),
            1 => Ok(Self::Bit32),
            2 => Ok(Self::Bit64),
            _ => Err(ParseError::InvalidClass(b)),
        }
    }
}

impl TryFrom<u16> for Type {
    type Error = ParseError;
    fn try_from(b: u16) -> Result<Self, Self::Error> {
        match b {
            0 => Ok(Self::None),
            1 => Ok(Self::Rel),
            2 => Ok(Self::Exec),
            3 => Ok(Self::Dyn),
            4 => Ok(Self::Core),
            _ => Err(ParseError::InvalidType(b)),
        }
    }
}

impl TryFrom<u16> for Machine {
    type Error = ParseError;
    fn try_from(b: u16) -> Result<Self, Self::Error> {
        match b {
            0 => Ok(Self::None),
            62 => Ok(Self::X86_64),
            183 => Ok(Self::AArch64),
            243 => Ok(Self::RiscV),
            _ => Err(ParseError::InvalidMachine(b)),
        }
    }
}

// マジックナンバーのパース
fn parse_magic_number(raw: &[u8]) -> ParseResult<()> {
    if raw.len() < 4 {
        return Err(nom::Err::Error(ParseError::InvalidHeaderSize(raw.len() as u8)));
    }
    if raw[..4] != ELF_MAGIC_NUMBER {
        let input: [u8; 4] = raw[..4].try_into().unwrap();
        return Err(nom::Err::Error(ParseError::FileTypeNotELF(input)));
    }
    Ok((&raw[4..], ()))
}

// ELF識別子のパース
fn parse_ident(raw: &[u8]) -> ParseResult<Ident> {
    let (rest, _) = parse_magic_number(raw)?;

    let (rest, class) = map_res(le_u8, Class::try_from).parse(rest)?;
    let (rest, data) = map_res(le_u8, Data::try_from).parse(rest)?;
    let (rest, version) = map_res(le_u8, IdentVersion::try_from).parse(rest)?;
    let (rest, os_abi) = map_res(le_u8, OSABI::try_from).parse(rest)?;
    let (rest, abi_version) = le_u8(rest)?;
    let (rest, _) = count(le_u8, 7).parse(rest)?; // パディング

    let ident = Ident {
        class,
        data,
        version,
        os_abi,
        abi_version,
    };
    Ok((rest, ident))
}

// ELFヘッダー全体のパース
pub fn parse(raw: &[u8]) -> ParseResult<Header> {
    if raw.len() < ELF_IDENT_HEADER_SIZE {
        return Err(nom::Err::Error(ParseError::InvalidHeaderSize(raw.len() as u8)));
    }

    let (rest, ident) = parse_ident(raw)?;
    let (rest, r#type) = map_res(le_u16, Type::try_from).parse(rest)?;
    let (rest, machine) = map_res(le_u16, Machine::try_from).parse(rest)?;
    let (rest, version) = map_res(le_u32, Version::try_from).parse(rest)?;
    let (rest, entry) = le_u64(rest)?;
    let (rest, phoff) = le_u64(rest)?;
    let (rest, shoff) = le_u64(rest)?;
    let (rest, flags) = le_u32(rest)?;
    let (rest, ehsize) = le_u16(rest)?;
    let (rest, phentsize) = le_u16(rest)?;
    let (rest, phnum) = le_u16(rest)?;
    let (rest, shentsize) = le_u16(rest)?;
    let (rest, shnum) = le_u16(rest)?;
    let (rest, shstrndx) = le_u16(rest)?;

    Ok((
        rest,
        Header {
            ident,
            r#type,
            machine,
            version,
            entry,
            phoff,
            shoff,
            flags,
            ehsize,
            phentsize,
            phnum,
            shentsize,
            shnum,
            shstrndx,
        },
    ))
}
```

## パーサーモジュールの定義

```rust:src/parser/mod.rs
pub mod error;
pub mod header;
pub mod helper;
pub mod relocation;
pub mod section;
pub mod symbol;

use error::ParseError;

pub type ParseResult<'a, T> = nom::IResult<&'a [u8], T, ParseError>;
```

## テスト

ELFヘッダーのパースが正しく動作するかテストする。

```rust:src/parser/header.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn should_error_invalid_magic_number() {
        let input: &[u8] = &[0u8; 16];
        let err = parse(input).unwrap_err();
        assert_eq!(
            err,
            nom::Err::Error(ParseError::FileTypeNotELF([0, 0, 0, 0]))
        );
    }

    #[test]
    fn should_parse_elf_header() {
        // sub.oのバイナリをテストデータとして使用
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

これでELFヘッダーのパースが実装できた。次章では、セクションヘッダーとシンボルテーブルのパースを実装する。
