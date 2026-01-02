---
title: "ELFパーサーの実装（3）再配置情報"
---

本章では、再配置情報のパースを実装し、ELFパーサー全体を統合する。

## 再配置エントリのデータ構造

再配置エントリのデータ構造を定義する。

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
pub struct RelocationAddend {
    pub offset: u64,
    pub info: Info,
    pub addend: i64,
}
```

`Info`は64ビットで、下位32ビットが再配置タイプ、上位32ビットがシンボルインデックスを表す。

## 再配置情報のパース

再配置情報をパースする関数を実装する。

```rust:src/parser/relocation.rs
use nom::{
    Parser as _,
    combinator::map_res,
    multi::count,
    number::complete::{le_i64, le_u64},
};
use super::{ParseResult, error::ParseError};
use crate::elf::{
    relocation::{Info, RelocationAddend, RelocationType},
    section,
};

impl TryFrom<u32> for RelocationType {
    type Error = ParseError;

    fn try_from(value: u32) -> Result<Self, Self::Error> {
        match value {
            274 => Ok(Self::Aarch64AdrPrelLo21),
            _ => Err(ParseError::InvalidRelocationType(value)),
        }
    }
}

impl TryFrom<u64> for Info {
    type Error = ParseError;

    fn try_from(value: u64) -> Result<Self, Self::Error> {
        // 下位32ビットが再配置タイプ
        let r#type = RelocationType::try_from((value & 0xffffffff) as u32)?;
        // 上位32ビットがシンボルインデックス
        let symbol_index = (value >> 32) as u32;
        Ok(Info {
            r#type,
            symbol_index,
        })
    }
}

fn parse_info(raw: &[u8]) -> ParseResult<Info> {
    map_res(le_u64, Info::try_from).parse(raw)
}

pub fn parse(section_headers: &[section::Header]) -> ParseResult<Vec<RelocationAddend>> {
    // .rela.textセクションを探す
    let Some(header) = section_headers
        .iter()
        .find(|&s| s.r#type == section::SectionType::Rela)
    else {
        return Ok((&[], vec![]));
    };

    let entry_count = (header.size / header.entsize) as usize;

    let (rest, relocations) = count(
        |raw| {
            let (rest, offset) = le_u64(raw)?;
            let (rest, info) = parse_info(rest)?;
            let (rest, addend) = le_i64(rest)?;

            let relocation = RelocationAddend {
                offset,
                info,
                addend,
            };

            Ok((rest, relocation))
        },
        entry_count,
    )
    .parse(header.section_raw_data.as_ref())?;

    Ok((rest, relocations))
}
```

## R_AARCH64_ADR_PREL_LO21について

本書で扱う再配置タイプは`R_AARCH64_ADR_PREL_LO21`（値274）のみである。

これはARM64の`ADR`命令で使用される再配置タイプで、PC（プログラムカウンタ）からの相対アドレスを21ビットの即値としてエンコードする。

`main.o`の再配置情報を確認すると次のようになっている。

```sh
$ readelf -r main.o
Relocation section '.rela.text' at offset 0x178 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000000  000900000112 R_AARCH64_ADR_PRE 0000000000000000 x + 0
```

- `Offset`: 0（`.text`セクションの先頭）
- `Info`: シンボルインデックス9、タイプ274
- `Sym. Name`: `x`
- `Addend`: 0

これは「`.text`セクションのオフセット0にある命令で、シンボル`x`への参照がある」ことを示している。

## ELFパーサーの統合

すべてのパース処理を統合し、ELF構造体を生成する関数を実装する。

```rust:src/parser.rs
pub mod error;
pub mod header;
pub mod helper;
pub mod relocation;
pub mod section;
pub mod symbol;

use crate::elf::{header::Header, relocation::RelocationAddend, section, symbol::Symbol};
use error::ParseError;

pub type ParseResult<'a, T> = nom::IResult<&'a [u8], T, ParseError>;

#[derive(Debug)]
pub struct ELF {
    pub header: Header,
    pub section_headers: Vec<section::Header>,
    pub symbols: Vec<Symbol>,
    pub relocations: Vec<RelocationAddend>,
}

pub fn parse_elf(raw: &[u8]) -> ParseResult<ELF> {
    // ELFヘッダーをパース
    let (_, header) = header::parse(raw)?;

    // セクションヘッダーをパース
    let (_, section_headers) = section::parse_header(
        raw,
        header.shoff as usize,
        header.shstrndx as usize,
        header.shnum as usize,
    )?;

    // シンボルテーブルをパース
    let (_, symbols) = symbol::parse(raw, &section_headers)?;

    // 再配置情報をパース
    let (_, relocations) = relocation::parse(&section_headers)?;

    Ok((
        &[],
        ELF {
            header,
            section_headers,
            symbols,
            relocations,
        },
    ))
}
```

## テスト

ELFパーサー全体のテストを実装する。

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn should_parse_elf() {
        let raw = include_bytes!("./parser/fixtures/main.o");
        let (_, elf) = parse_elf(raw).unwrap();

        // ELFヘッダーの確認
        assert_eq!(elf.header.r#type, crate::elf::header::Type::Rel);
        assert_eq!(elf.header.machine, crate::elf::header::Machine::AArch64);

        // シンボルの確認
        let start_symbol = elf.symbols.iter().find(|s| s.name == "_start");
        assert!(start_symbol.is_some());

        let x_symbol = elf.symbols.iter().find(|s| s.name == "x");
        assert!(x_symbol.is_some());

        // 再配置の確認
        assert_eq!(elf.relocations.len(), 1);
        assert_eq!(
            elf.relocations[0].info.r#type,
            crate::elf::relocation::RelocationType::Aarch64AdrPrelLo21
        );
    }
}
```

これでELFパーサーの実装が完了した。次章では、リンク処理の仕組みを解説する。
