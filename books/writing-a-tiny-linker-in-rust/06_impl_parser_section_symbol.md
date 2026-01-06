---
title: "ELFパーサーの実装（2）セクションとシンボル"
---

本章では、セクションヘッダーとシンボルテーブルのパースを実装する。

## 本章で実装するファイル

```
src/
├── elf/
│   ├── section.rs    # 本章
│   └── symbol.rs     # 本章
├── parser/
│   ├── error.rs      # 本章（エラー追加）
│   ├── helper.rs     # 本章
│   ├── section.rs    # 本章
│   └── symbol.rs     # 本章
└── ...
```

## モジュール構造を更新する

LSPが正しく動作するように、最初にモジュール宣言を追加する。

### elf.rsを更新する

```diff:src/elf.rs
 pub mod header;
+pub mod section;
+pub mod symbol;
```

### parser.rsを更新する

```diff:src/parser.rs
 pub mod error;
 pub mod header;
+pub mod helper;
+pub mod section;
+pub mod symbol;

 use error::ParseError;

 pub type ParseResult<'a, T> = nom::IResult<&'a [u8], T, ParseError>;
```

### 空のモジュールファイルを作成する

```sh
$ touch src/elf/section.rs src/elf/symbol.rs
$ touch src/parser/helper.rs src/parser/section.rs src/parser/symbol.rs
```

## セクションヘッダーのパース

### データ構造を定義する

`src/elf/section.rs`を実装する。

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
    NoBits = 8,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
#[repr(u64)]
pub enum SectionFlag {
    #[default]
    None = 0,
    Write = 0x1,
    Alloc = 0x2,
    ExecInstr = 0x4,
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
    pub data: Vec<u8>,
}
```

### 文字列テーブルのヘルパー関数を実装する

セクション名やシンボル名は文字列テーブルに格納されている。オフセットからNULL終端の文字列を取得するヘルパー関数を実装する。

`src/parser/helper.rs`を実装する。

```rust:src/parser/helper.rs
pub fn get_string(data: &[u8], offset: usize) -> String {
    let mut end = offset;
    while end < data.len() && data[end] != 0 {
        end += 1;
    }
    String::from_utf8_lossy(&data[offset..end]).to_string()
}
```

### パーサーエラーを追加する

```diff:src/parser/error.rs
     #[error("Invalid version: {0}")]
     InvalidVersion(u32),
+
+    #[error("Invalid section type: {0}")]
+    InvalidSectionType(u32),
 }
```

### テストを書く

`src/parser/section.rs`にテストを書く。

```rust:src/parser/section.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::elf::section::SectionType;

    #[test]
    fn parse_section_headers() {
        let raw = include_bytes!("./fixtures/sub.o");
        let (_, elf_header) = crate::parser::header::parse(raw).unwrap();
        let (_, headers) = parse_headers(
            raw,
            elf_header.shoff as usize,
            elf_header.shstrndx as usize,
            elf_header.shnum as usize,
        )
        .unwrap();

        // セクション数の確認
        assert_eq!(headers.len(), 9);

        // .textセクションの確認
        let text = headers.iter().find(|h| h.name == ".text").unwrap();
        assert_eq!(text.r#type, SectionType::ProgBits);

        // .dataセクションの確認
        let data = headers.iter().find(|h| h.name == ".data").unwrap();
        assert_eq!(data.r#type, SectionType::ProgBits);
        assert_eq!(data.size, 4);

        // .symtabセクションの確認
        let symtab = headers.iter().find(|h| h.name == ".symtab").unwrap();
        assert_eq!(symtab.r#type, SectionType::SymTab);
    }
}
```

### パース処理を実装する

```diff:src/parser/section.rs
+use super::{helper, ParseResult};
+use crate::elf::section::{Header, SectionFlag, SectionType};
+use crate::parser::error::ParseError;
+use nom::combinator::map_res;
+use nom::multi::count;
+use nom::number::complete::{le_u32, le_u64};
+use nom::Parser as _;
+
+impl TryFrom<u32> for SectionType {
+    type Error = ParseError;
+    fn try_from(value: u32) -> Result<Self, Self::Error> {
+        match value {
+            0 => Ok(Self::Null),
+            1 => Ok(Self::ProgBits),
+            2 => Ok(Self::SymTab),
+            3 => Ok(Self::StrTab),
+            4 => Ok(Self::Rela),
+            8 => Ok(Self::NoBits),
+            _ => Err(ParseError::InvalidSectionType(value)),
+        }
+    }
+}
+
+fn parse_flags(raw: &[u8]) -> ParseResult<'_, Vec<SectionFlag>> {
+    let (rest, mask) = le_u64(raw)?;
+    let mut flags = vec![];
+    if mask & 0x1 != 0 {
+        flags.push(SectionFlag::Write);
+    }
+    if mask & 0x2 != 0 {
+        flags.push(SectionFlag::Alloc);
+    }
+    if mask & 0x4 != 0 {
+        flags.push(SectionFlag::ExecInstr);
+    }
+    Ok((rest, flags))
+}
+
+pub fn parse_headers(
+    raw: &[u8],
+    shoff: usize,
+    shstrndx: usize,
+    shnum: usize,
+) -> ParseResult<'_, Vec<Header>> {
+    if shnum == 0 {
+        return Ok((raw, vec![]));
+    }
+
+    let (rest, mut headers) = count(
+        |input| {
+            let (rest, name_idx) = le_u32(input)?;
+            let (rest, r#type) = map_res(le_u32, SectionType::try_from).parse(rest)?;
+            let (rest, flags) = parse_flags(rest)?;
+            let (rest, addr) = le_u64(rest)?;
+            let (rest, offset) = le_u64(rest)?;
+            let (rest, size) = le_u64(rest)?;
+            let (rest, link) = le_u32(rest)?;
+            let (rest, info) = le_u32(rest)?;
+            let (rest, addralign) = le_u64(rest)?;
+            let (rest, entsize) = le_u64(rest)?;
+
+            let data = if size > 0 && r#type != SectionType::NoBits {
+                raw[offset as usize..(offset + size) as usize].to_vec()
+            } else {
+                vec![]
+            };
+
+            let header = Header {
+                name_idx,
+                name: String::new(),
+                r#type,
+                flags,
+                addr,
+                offset,
+                size,
+                link,
+                info,
+                addralign,
+                entsize,
+                data,
+            };
+            Ok((rest, header))
+        },
+        shnum,
+    )
+    .parse(&raw[shoff..])?;
+
+    // セクション名を文字列テーブルから解決
+    let shstrtab = &headers[shstrndx];
+    let strtab_data = &raw[shstrtab.offset as usize..(shstrtab.offset + shstrtab.size) as usize];
+    for header in headers.iter_mut() {
+        header.name = helper::get_string(strtab_data, header.name_idx as usize);
+    }
+
+    Ok((rest, headers))
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test parser::section
running 1 test
test parser::section::tests::parse_section_headers ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## シンボルテーブルのパース

### データ構造を定義する

`src/elf/symbol.rs`を実装する。

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
pub enum SymbolType {
    #[default]
    NoType = 0,
    Object = 1,
    Func = 2,
    Section = 3,
    File = 4,
}

#[derive(Default, Debug, PartialEq, Eq, Clone, Copy)]
pub struct Info {
    pub r#type: SymbolType,
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

/// 未定義シンボルのセクションインデックス
pub const SHN_UNDEF: u16 = 0;
```

### パーサーエラーを追加する

```diff:src/parser/error.rs
     #[error("Invalid section type: {0}")]
     InvalidSectionType(u32),
+
+    #[error("Invalid symbol binding: {0}")]
+    InvalidSymbolBinding(u8),
+
+    #[error("Invalid symbol type: {0}")]
+    InvalidSymbolType(u8),
+
+    #[error("Invalid visibility: {0}")]
+    InvalidVisibility(u8),
 }
```

### テストを書く

`src/parser/symbol.rs`にテストを書く。

```rust:src/parser/symbol.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::elf::symbol::{Binding, SymbolType};

    #[test]
    fn parse_symbols() {
        let raw = include_bytes!("./fixtures/sub.o");
        let (_, elf_header) = crate::parser::header::parse(raw).unwrap();
        let (_, section_headers) = crate::parser::section::parse_headers(
            raw,
            elf_header.shoff as usize,
            elf_header.shstrndx as usize,
            elf_header.shnum as usize,
        )
        .unwrap();

        let (_, symbols) = parse(raw, &section_headers).unwrap();

        // 変数xのシンボルを確認
        let x = symbols.iter().find(|s| s.name == "x").unwrap();
        assert_eq!(x.info.binding, Binding::Global);
        assert_eq!(x.info.r#type, SymbolType::Object);
        assert_eq!(x.size, 4);
    }
}
```

### パース処理を実装する

```diff:src/parser/symbol.rs
+use super::{helper, ParseResult};
+use crate::elf::section::{Header, SectionType};
+use crate::elf::symbol::{Binding, Info, Symbol, SymbolType, Visibility};
+use crate::parser::error::ParseError;
+use nom::combinator::map_res;
+use nom::multi::count;
+use nom::number::complete::{le_u16, le_u32, le_u64, le_u8};
+use nom::Parser as _;
+
+impl TryFrom<u8> for Info {
+    type Error = ParseError;
+    fn try_from(value: u8) -> Result<Self, Self::Error> {
+        let binding = match value >> 4 {
+            0 => Binding::Local,
+            1 => Binding::Global,
+            2 => Binding::Weak,
+            _ => return Err(ParseError::InvalidSymbolBinding(value)),
+        };
+        let r#type = match value & 0xf {
+            0 => SymbolType::NoType,
+            1 => SymbolType::Object,
+            2 => SymbolType::Func,
+            3 => SymbolType::Section,
+            4 => SymbolType::File,
+            _ => return Err(ParseError::InvalidSymbolType(value)),
+        };
+        Ok(Info { r#type, binding })
+    }
+}
+
+impl TryFrom<u8> for Visibility {
+    type Error = ParseError;
+    fn try_from(value: u8) -> Result<Self, Self::Error> {
+        match value {
+            0 => Ok(Self::Default),
+            1 => Ok(Self::Internal),
+            2 => Ok(Self::Hidden),
+            3 => Ok(Self::Protected),
+            _ => Err(ParseError::InvalidVisibility(value)),
+        }
+    }
+}
+
+pub fn parse<'a>(raw: &'a [u8], section_headers: &[Header]) -> ParseResult<'a, Vec<Symbol>> {
+    // .symtabセクションを探す
+    let Some(symtab) = section_headers
+        .iter()
+        .find(|h| h.r#type == SectionType::SymTab)
+    else {
+        return Ok((&[], vec![]));
+    };
+
+    // .strtab（シンボル名の文字列テーブル）を取得
+    let strtab = &section_headers[symtab.link as usize];
+    let strtab_data = &raw[strtab.offset as usize..(strtab.offset + strtab.size) as usize];
+
+    let entry_count = (symtab.size / symtab.entsize) as usize;
+
+    let (rest, symbols) = count(
+        |input| {
+            let (rest, name_idx) = le_u32(input)?;
+            let (rest, info) = map_res(le_u8, Info::try_from).parse(rest)?;
+            let (rest, other) = map_res(le_u8, Visibility::try_from).parse(rest)?;
+            let (rest, shndx) = le_u16(rest)?;
+            let (rest, value) = le_u64(rest)?;
+            let (rest, size) = le_u64(rest)?;
+
+            let name = helper::get_string(strtab_data, name_idx as usize);
+            let symbol = Symbol {
+                name,
+                info,
+                other,
+                shndx,
+                value,
+                size,
+            };
+            Ok((rest, symbol))
+        },
+        entry_count,
+    )
+    .parse(&symtab.data)?;
+
+    Ok((rest, symbols))
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test parser::symbol
running 1 test
test parser::symbol::tests::parse_symbols ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## まとめ

本章ではセクションヘッダーとシンボルテーブルのパースを実装した。

- モジュール宣言を先に追加してLSPを有効化
- セクションヘッダーの`link`フィールドで関連する文字列テーブルを参照
- シンボルの`info`フィールドは上位4ビットがバインディング、下位4ビットがタイプ
- 文字列テーブルからオフセットを使って名前を解決

次章では、再配置情報のパースを実装する。
