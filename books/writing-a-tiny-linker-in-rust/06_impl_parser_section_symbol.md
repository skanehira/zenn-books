---
title: "ELFパーサーの実装（2）セクションとシンボル"
---

本章では、セクションヘッダーとシンボルテーブルのパースを実装する。

## セクションヘッダーのデータ構造

まず、セクションヘッダーのデータ構造を定義する。

```rust:src/elf/section.rs
#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u32)]
pub enum SectionType {
    #[default]
    Null = 0,
    ProgBits = 1,
    SymTab = 2,
    StrTab = 3,
    Rela = 4,
    Hash = 5,
    Dynamic = 6,
    Note = 7,
    NoBits = 8,
    Rel = 9,
    ShLib = 10,
    DynSym = 11,
    InitArray = 14,
    FiniArray = 15,
    PreInitArray = 16,
    Group = 17,
    SymTabShndx = 18,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u64)]
pub enum SectionFlag {
    #[default]
    None = 0,
    Write = 0x1,
    Alloc = 0x2,
    ExecInstr = 0x4,
    Merge = 0x10,
    Strings = 0x20,
    InfoLink = 0x40,
    LinkOrder = 0x80,
    Group = 0x200,
    Tls = 0x400,
    Compressed = 0x800,
}

#[derive(Default, Debug, PartialEq, Eq, Clone)]
pub struct Header {
    pub name_idx: u32,
    pub name: String,
    pub r#type: SectionType,
    pub flags: Vec<SectionFlag>,
    pub addr: u64,
    pub offset: u64,
    pub size: u64,
    pub link: u32,
    pub info: u32,
    pub addralign: u64,
    pub entsize: u64,
    pub section_raw_data: Vec<u8>,
}
```

## 文字列テーブルからの名前解決

セクション名やシンボル名は文字列テーブルに格納されており、オフセットで参照される。
NULL終端の文字列を取得するヘルパー関数を実装する。

```rust:src/parser/helper.rs
pub fn get_string_by_offset(raw: &[u8], offset: usize) -> String {
    let mut end = offset;
    while end < raw.len() && raw[end] != 0 {
        end += 1;
    }
    String::from_utf8_lossy(&raw[offset..end]).to_string()
}
```

## セクションヘッダーのパース

セクションヘッダーをパースする関数を実装する。

```rust:src/parser/section.rs
use crate::elf::section::{Header, SectionFlag, SectionType};
use crate::parser::error::ParseError;
use crate::parser::helper;
use nom::{
    Parser as _,
    combinator::map_res,
    multi::count,
    number::complete::{le_u32, le_u64},
};
use super::ParseResult;

impl TryFrom<u32> for SectionType {
    type Error = ParseError;
    fn try_from(value: u32) -> Result<Self, Self::Error> {
        match value {
            0 => Ok(Self::Null),
            1 => Ok(Self::ProgBits),
            2 => Ok(Self::SymTab),
            3 => Ok(Self::StrTab),
            4 => Ok(Self::Rela),
            8 => Ok(Self::NoBits),
            // ... 他のタイプ
            _ => Err(ParseError::InvalidSectionType(value)),
        }
    }
}

fn parse_type(raw: &[u8]) -> ParseResult<SectionType> {
    map_res(le_u32, SectionType::try_from).parse(raw)
}

fn parse_flags(raw: &[u8]) -> ParseResult<Vec<SectionFlag>> {
    map_res(le_u64, |mask| -> Result<Vec<SectionFlag>, ParseError> {
        let flag_variants = [
            SectionFlag::Write,
            SectionFlag::Alloc,
            SectionFlag::ExecInstr,
            SectionFlag::Merge,
            SectionFlag::Strings,
            SectionFlag::InfoLink,
            SectionFlag::LinkOrder,
            SectionFlag::Group,
            SectionFlag::Tls,
            SectionFlag::Compressed,
        ];

        if mask == 0 {
            return Ok(vec![]);
        }

        let flags = flag_variants
            .into_iter()
            .filter(|flag| mask & (*flag as u64) != 0)
            .collect();

        Ok(flags)
    })
    .parse(raw)
}

pub fn parse_header(
    raw: &[u8],
    shoff: usize,
    shstrndx: usize,
    shnum: usize,
) -> ParseResult<Vec<Header>> {
    if shnum == 0 {
        return Ok((raw, vec![]));
    }

    count(
        |rest| {
            let (rest, name_idx) = le_u32(rest)?;
            let (rest, r#type) = parse_type(rest)?;
            let (rest, flags) = parse_flags(rest)?;
            let (rest, addr) = le_u64(rest)?;
            let (rest, offset) = le_u64(rest)?;
            let (rest, size) = le_u64(rest)?;
            let (rest, link) = le_u32(rest)?;
            let (rest, info) = le_u32(rest)?;
            let (rest, addralign) = le_u64(rest)?;
            let (rest, entsize) = le_u64(rest)?;

            // セクションのデータを取得
            let data = raw[offset as usize..(offset + size) as usize].to_vec();

            let header = Header {
                name_idx,
                name: "".into(), // 後で埋める
                r#type,
                flags,
                addr,
                offset,
                size,
                link,
                info,
                addralign,
                entsize,
                section_raw_data: data,
            };

            Ok((rest, header))
        },
        shnum,
    )
    .parse(&raw[shoff..])
    .map(|(rest, mut headers)| {
        // セクション名を文字列テーブルから解決
        let shstrtab = &headers[shstrndx];
        let string_table =
            &raw[shstrtab.offset as usize..(shstrtab.offset + shstrtab.size) as usize];
        for header in headers.iter_mut() {
            header.name = helper::get_string_by_offset(string_table, header.name_idx as usize);
        }
        (rest, headers)
    })
}
```

ポイントは次のとおり。

1. ELFヘッダーの`shoff`、`shstrndx`、`shnum`を使ってセクションヘッダーの位置と数を特定
2. `count`コンビネータで`shnum`回繰り返しパース
3. パース後、`.shstrtab`（セクションヘッダー文字列テーブル）からセクション名を解決

## シンボルテーブルのデータ構造

シンボルのデータ構造を定義する。

```rust:src/elf/symbol.rs
#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u8)]
pub enum Binding {
    #[default]
    Local = 0,
    Global = 1,
    Weak = 2,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u8)]
pub enum Type {
    #[default]
    NoType = 0,
    Object = 1,
    Func = 2,
    Section = 3,
    File = 4,
    Common = 5,
    Tls = 6,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
pub struct Info {
    pub r#type: Type,
    pub binding: Binding,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u8)]
pub enum Visibility {
    #[default]
    Default = 0,
    Internal = 1,
    Hidden = 2,
    Protected = 3,
}

#[derive(Default, Debug, PartialEq, Eq, Clone)]
pub struct Symbol {
    pub name: String,
    pub info: Info,
    pub other: Visibility,
    pub shndx: u16,
    pub value: u64,
    pub size: u64,
}

/// 未定義シンボルのインデックス
pub const SHN_UNDEF: u16 = 0;
```

`Info`は1バイトに上位4ビットのバインディングと下位4ビットのタイプが詰め込まれている。

## シンボルテーブルのパース

シンボルテーブルをパースする関数を実装する。

```rust:src/parser/symbol.rs
use super::{ParseResult, error::ParseError, helper};
use crate::elf::{
    section::{Header, SectionType},
    symbol::{Binding, Info, Symbol, Type, Visibility},
};
use nom::{
    Parser as _,
    combinator::map_res,
    multi::count,
    number::complete::{le_u8, le_u16, le_u64, le_u32},
};

impl TryFrom<u8> for Info {
    type Error = ParseError;
    fn try_from(value: u8) -> Result<Self, Self::Error> {
        // 上位4ビット: バインディング
        let binding = match value >> 4 {
            0 => Binding::Local,
            1 => Binding::Global,
            2 => Binding::Weak,
            _ => return Err(ParseError::InvalidSymbolBinding(value)),
        };

        // 下位4ビット: タイプ
        let r#type = match value & 0xf {
            0 => Type::NoType,
            1 => Type::Object,
            2 => Type::Func,
            3 => Type::Section,
            4 => Type::File,
            5 => Type::Common,
            6 => Type::Tls,
            _ => return Err(ParseError::InvalidSymbolType(value)),
        };

        Ok(Self { r#type, binding })
    }
}

impl TryFrom<u8> for Visibility {
    type Error = ParseError;
    fn try_from(b: u8) -> Result<Self, Self::Error> {
        match b {
            0 => Ok(Self::Default),
            1 => Ok(Self::Internal),
            2 => Ok(Self::Hidden),
            3 => Ok(Self::Protected),
            _ => Err(ParseError::InvalidVisibility(b)),
        }
    }
}

pub fn parse<'a>(raw: &'a [u8], section_headers: &'a [Header]) -> ParseResult<'a, Vec<Symbol>> {
    // .symtabセクションを探す
    let Some(symbol_header) = section_headers
        .iter()
        .find(|header| header.r#type == SectionType::SymTab)
    else {
        return Ok((&[], vec![]));
    };

    // .strtabを取得（シンボル名の文字列テーブル）
    let strtab = &section_headers[symbol_header.link as usize];
    let string_table = &raw[strtab.offset as usize..(strtab.offset + strtab.size) as usize];

    // シンボルエントリの数を計算
    let entry_count = (symbol_header.size / symbol_header.entsize) as usize;

    let (rest, symbols) = count(
        |raw| {
            let (rest, name_idx) = le_u32(raw)?;
            let (rest, info) = map_res(le_u8, Info::try_from).parse(rest)?;
            let (rest, other) = map_res(le_u8, Visibility::try_from).parse(rest)?;
            let (rest, shndx) = le_u16(rest)?;
            let (rest, value) = le_u64(rest)?;
            let (rest, size) = le_u64(rest)?;

            // シンボル名を文字列テーブルから取得
            let name = helper::get_string_by_offset(string_table, name_idx as usize);
            let symbol = Symbol {
                name,
                info,
                other,
                shndx,
                value,
                size,
            };

            Ok((rest, symbol))
        },
        entry_count,
    )
    .parse(symbol_header.section_raw_data.as_ref())?;

    Ok((rest, symbols))
}
```

ポイントは次のとおり。

1. セクションヘッダーから`.symtab`（シンボルテーブル）を探す
2. シンボルテーブルの`link`フィールドが指す`.strtab`（文字列テーブル）を取得
3. `size / entsize`でシンボルエントリの数を計算
4. 各シンボルをパースし、名前を文字列テーブルから解決

## テスト

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_symbol_ok() {
        let raw = include_bytes!("./fixtures/sub.o");
        let (_, header) = crate::parser::header::parse(raw).unwrap();
        let (_, section_headers) = crate::parser::section::parse_header(
            raw,
            header.shoff as usize,
            header.shstrndx as usize,
            header.shnum as usize,
        )
        .unwrap();

        let symbols = parse(raw, &section_headers).unwrap().1;

        // 最後のシンボルが変数xであることを確認
        let x_symbol = symbols.last().unwrap();
        assert_eq!(x_symbol.name, "x");
        assert_eq!(x_symbol.info.binding, Binding::Global);
        assert_eq!(x_symbol.info.r#type, Type::Object);
        assert_eq!(x_symbol.size, 4);
    }
}
```

これでセクションヘッダーとシンボルテーブルのパースが実装できた。次章では、再配置情報のパースを実装する。
