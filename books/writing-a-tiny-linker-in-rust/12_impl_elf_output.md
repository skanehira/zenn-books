---
title: "ELF出力の実装"
---

本章では、ここまでで作ってきたセクションたちを、実行可能なELFファイルとして書き出す処理を実装する。

## 本章で実装するファイル

```
src/
├── elf.rs             # 本章（モジュール追加）
├── elf/
│   ├── header.rs         # 本章（to_bytes追加）
│   ├── program_header.rs # 本章
│   └── segment.rs        # 本章
├── linker.rs          # 本章（モジュール追加）
├── linker/
│   └── writer.rs         # 本章
└── ...
```

## モジュール構造を更新する

### elf.rsを更新する

```diff:src/elf.rs
 pub mod header;
+pub mod program_header;
 pub mod relocation;
 pub mod section;
+pub mod segment;
 pub mod symbol;
```

### linker.rsを更新する

```diff:src/linker.rs
 pub mod output;
 pub mod relocation;
 pub mod section;
 pub mod symbol;
+pub mod writer;

 use crate::parser::Elf;
```

### 空のモジュールファイルを作成する

```sh
$ touch src/elf/program_header.rs src/elf/segment.rs
$ touch src/linker/writer.rs
```

## プログラムヘッダー

ELFには「セクション」とは別に「セグメント」という概念がある。
セグメントは実行時にOSがメモリにロードする単位で、それぞれに読み取り・書き込み・実行の権限が付く。

セグメント自体は実行時の概念で、ELFファイルの中ではプログラムヘッダー（Program Header）としてそのメタデータが記録される。

### セグメントの種類と権限フラグ

`src/elf/segment.rs`にセグメントの`Type`と`Flag`を定義する。

```rust:src/elf/segment.rs
#[derive(Debug, Clone, Copy)]
#[repr(u32)]
pub enum Type {
    Load = 1,
}

#[derive(Debug, Clone, Copy)]
#[repr(u32)]
pub enum Flag {
    Executable = 0x1,
    Writable = 0x2,
    Readable = 0x4,
}
```

### プログラムヘッダー構造体

`src/elf/program_header.rs`に`ProgramHeader`構造体を追加する。

```rust:src/elf/program_header.rs
use super::segment::{Flag, Type};

#[derive(Debug, Clone)]
pub struct ProgramHeader {
    pub r#type: Type,
    pub flags: Vec<Flag>,
    pub offset: u64,
    pub vaddr: u64,
    pub paddr: u64,
    pub filesz: u64,
    pub memsz: u64,
    pub align: u64,
}
```

## ELFヘッダーのシリアライズ

ELFヘッダーは構造体としてはすでに作ってあるが、ファイルに書き出すためにバイト列へのシリアライズが必要になるので、`to_bytes`メソッドとして追加する。

### テストを書く

```diff:src/elf/header.rs
 impl Header {
+    pub fn to_bytes(&self) -> Vec<u8> {
+        todo!()
+    }
 }
```

```diff:src/elf/header.rs
 #[cfg(test)]
 mod tests {
+    use super::*;
+
+    #[test]
+    fn header_to_bytes_returns_64_bytes() {
+        let header = Header {
+            ident: Ident {
+                class: Class::Bit64,
+                data: Data::Lsb,
+                version: IdentVersion::Current,
+                os_abi: OSABI::SystemV,
+                abi_version: 0,
+            },
+            r#type: Type::Exec,
+            machine: Machine::AArch64,
+            version: Version::Current,
+            entry: 0x400100,
+            phoff: 64,
+            shoff: 0x1e0,
+            flags: 0,
+            ehsize: 64,
+            phentsize: 56,
+            phnum: 2,
+            shentsize: 64,
+            shnum: 6,
+            shstrndx: 5,
+        };
+
+        let bytes = header.to_bytes();
+        assert_eq!(bytes.len(), 64);
+
+        // マジックナンバー
+        assert_eq!(&bytes[0..4], &[0x7f, b'E', b'L', b'F']);
+    }
 }
```

### to_bytesメソッドを実装する

パース時の逆をやるだけで、構造体定義の順番どおりに値を書き出していく。

```diff:src/elf/header.rs
 impl Header {
     pub fn to_bytes(&self) -> Vec<u8> {
-        todo!()
+        let mut bytes = Vec::with_capacity(64);
+
+        bytes.extend_from_slice(&[0x7f, b'E', b'L', b'F']); // マジック
+        bytes.push(self.ident.class as u8);
+        bytes.push(self.ident.data as u8);
+        bytes.push(self.ident.version as u8);
+        bytes.push(self.ident.os_abi as u8);
+        bytes.push(self.ident.abi_version);
+        bytes.extend_from_slice(&[0u8; 7]); // パディング
+
+        bytes.extend_from_slice(&(self.r#type as u16).to_le_bytes());
+        bytes.extend_from_slice(&(self.machine as u16).to_le_bytes());
+        bytes.extend_from_slice(&(self.version as u32).to_le_bytes());
+        bytes.extend_from_slice(&self.entry.to_le_bytes());
+        bytes.extend_from_slice(&self.phoff.to_le_bytes());
+        bytes.extend_from_slice(&self.shoff.to_le_bytes());
+        bytes.extend_from_slice(&self.flags.to_le_bytes());
+        bytes.extend_from_slice(&self.ehsize.to_le_bytes());
+        bytes.extend_from_slice(&self.phentsize.to_le_bytes());
+        bytes.extend_from_slice(&self.phnum.to_le_bytes());
+        bytes.extend_from_slice(&self.shentsize.to_le_bytes());
+        bytes.extend_from_slice(&self.shnum.to_le_bytes());
+        bytes.extend_from_slice(&self.shstrndx.to_le_bytes());
+
+        bytes
     }
 }
```

### テストを実行する

```sh
$ cargo test elf::header::tests::header_to_bytes
running 1 test
test elf::header::tests::header_to_bytes_returns_64_bytes ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## ELF出力の実装

部品がそろったので、いよいよELFファイル全体を書き出す処理を実装する。
書き出す順番はELFヘッダー → プログラムヘッダー → セクションデータ → セクションヘッダーで、ファイル内のオフセットとアドレスを意識しながら順に流し込んでいく。

### テストを書く

```rust:src/linker/writer.rs
use std::collections::HashMap;
use std::io::{Seek, Write};

use crate::error::Result;

use super::output::{ResolvedSymbol, Section};
use super::Linker;

impl Linker {
    pub fn write_executable<W: Write + Seek>(
        &self,
        _writer: &mut W,
        _resolved_symbols: HashMap<String, ResolvedSymbol>,
        _section_tables: Vec<Section<'static>>,
        _section_name_offsets: HashMap<String, usize>,
    ) -> Result<()> {
        todo!()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::linker::Linker;
    use crate::parser::Elf;
    use std::io::Cursor;

    #[test]
    fn write_executable_writes_valid_elf_header() {
        let main_o = include_bytes!("../parser/fixtures/main.o");
        let sub_o = include_bytes!("../parser/fixtures/sub.o");

        let mut linker = Linker::default();
        linker.add_object("main.o".to_string(), Elf::parse(main_o).unwrap());
        linker.add_object("sub.o".to_string(), Elf::parse(sub_o).unwrap());

        let mut resolved = linker.resolve_symbols().unwrap();
        let (sections, name_offsets) = linker.layout_sections(&mut resolved).unwrap();

        let mut output = Cursor::new(Vec::new());
        linker
            .write_executable(&mut output, resolved, sections, name_offsets)
            .unwrap();

        let bytes = output.into_inner();

        // ELFマジックナンバー
        assert_eq!(&bytes[0..4], &[0x7f, b'E', b'L', b'F']);

        // ファイルタイプ（EXEC = 2）
        assert_eq!(bytes[16], 2);
        assert_eq!(bytes[17], 0);

        // マシン（AArch64 = 0xb7）
        assert_eq!(bytes[18], 0xb7);
        assert_eq!(bytes[19], 0);
    }
}
```

### ELF出力を実装する

#### .symtabセクションヘッダの規約

実装に入る前に、`.symtab` のセクションヘッダだけ少し特殊なので先に触れておく。
`.symtab` セクションのヘッダだけは、他のセクションと違って `sh_link` / `sh_info` / `sh_entsize`
の各フィールドに **意味のある値** を入れる必要がある。System V gABIで次のように規定されている。

| フィールド | 意味 | 本書での値 |
| --- | --- | --- |
| `sh_link` | シンボル名を引く文字列テーブルセクションのインデックス | `.strtab` のセクションヘッダインデックス |
| `sh_info` | 最後のLOCALシンボルのインデックス + 1（= GLOBALの開始位置） | LOCALシンボル数 + 1（先頭のNULL分） |
| `sh_entsize` | 1エントリのサイズ | ELF64では24 |

`sh_info` の「LOCALの個数 + 1」になるのは、シンボルテーブルの先頭に必ずNULLシンボルを
1個置く規約があるため。つまりインデックス`[0]`がNULL、`[1..sh_info]`がLOCAL、
`[sh_info..]`がGLOBALという並びになる。

`readelf -s` で `.symtab` を表示するときも、リンカや動的ローダがシンボルを引くときも、
この `sh_link` から `.strtab` を辿ってシンボル名を取得する。これらを正しくセットしないと
`readelf -s` で `<corrupt: ...>` のような表示になるので、出力後の検証で気づきやすい。

実装は次のとおり。

```diff:src/linker/writer.rs
 use std::collections::HashMap;
-use std::io::{Seek, Write};
+use std::io::{Seek, SeekFrom, Write};

-use crate::error::Result;
+use crate::elf::header::{self, Class, Data, Ident, IdentVersion, Machine, OSABI, Type, Version};
+use crate::elf::program_header::ProgramHeader;
+use crate::elf::segment::{Flag, Type as SegmentType};
+use crate::elf::symbol::Binding;
+use crate::error::{LinkerError, Result};

 use super::output::{ResolvedSymbol, Section};
+use super::section::{align, BASE_ADDR};
 use super::Linker;

 impl Linker {
     pub fn write_executable<W: Write + Seek>(
         &self,
-        _writer: &mut W,
-        _resolved_symbols: HashMap<String, ResolvedSymbol>,
-        _section_tables: Vec<Section<'static>>,
-        _section_name_offsets: HashMap<String, usize>,
+        writer: &mut W,
+        resolved_symbols: HashMap<String, ResolvedSymbol>,
+        section_tables: Vec<Section<'static>>,
+        section_name_offsets: HashMap<String, usize>,
     ) -> Result<()> {
-        todo!()
+        // エントリポイント（_start）のアドレスを取得
+        let entry = resolved_symbols
+            .get("_start")
+            .ok_or(LinkerError::MissingEntryPoint)?
+            .value;
+
+        // ELFヘッダーを書き出し
+        let elf_header = self.create_elf_header(entry, &section_tables);
+        writer.write_all(&elf_header.to_bytes())?;
+
+        // プログラムヘッダーを書き出し
+        let program_headers = self.create_program_headers(&section_tables);
+        self.write_program_headers(writer, &program_headers)?;
+
+        // セクションデータを書き出し
+        self.write_sections(writer, &section_tables)?;
+
+        // セクションヘッダーテーブルの位置に移動
+        writer.seek(SeekFrom::Start(elf_header.shoff))?;
+
+        // NULLセクションヘッダー
+        self.write_section_header(writer, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)?;
+
+        // 各セクションのヘッダー
+        for section in section_tables.iter() {
+            let name_offset = *section_name_offsets
+                .get(section.name.as_ref())
+                .unwrap_or(&0);
+
+            let mut sh_flags: u64 = 0;
+            for f in &section.flags {
+                sh_flags |= *f as u64;
+            }
+
+            // .symtabの場合は特別な処理
+            let (sh_link, sh_info) = if section.name == ".symtab" {
+                let strtab_idx = section_tables
+                    .iter()
+                    .position(|s| s.name == ".strtab")
+                    .map(|i| i + 1)
+                    .unwrap_or(0) as u32;
+
+                let local_count = resolved_symbols
+                    .values()
+                    .filter(|s| s.info.binding == Binding::Local)
+                    .count() as u32
+                    + 1;
+
+                (strtab_idx, local_count)
+            } else {
+                (0, 0)
+            };
+
+            let sh_entsize = if section.name == ".symtab" { 24 } else { 0 };
+
+            self.write_section_header(
+                writer,
+                name_offset as u32,
+                section.r#type as u32,
+                sh_flags,
+                section.addr,
+                section.offset,
+                section.size,
+                sh_link,
+                sh_info,
+                section.align,
+                sh_entsize,
+            )?;
+        }
+
+        Ok(())
+    }
+
+    fn create_elf_header(
+        &self,
+        entry: u64,
+        section_tables: &[Section<'static>],
+    ) -> header::Header {
+        let shoff = section_tables
+            .iter()
+            .map(|s| s.offset + align(s.size, 8))
+            .max()
+            .unwrap();
+
+        let shnum = (section_tables.len() + 1) as u16;
+
+        let shstrndx = section_tables
+            .iter()
+            .position(|s| s.name == ".shstrtab")
+            .map(|i| (i + 1) as u16)
+            .unwrap_or(0);
+
+        header::Header {
+            ident: Ident {
+                class: Class::Bit64,
+                data: Data::Lsb,
+                version: IdentVersion::Current,
+                os_abi: OSABI::SystemV,
+                abi_version: 0,
+            },
+            r#type: Type::Exec,
+            machine: Machine::AArch64,
+            version: Version::Current,
+            entry,
+            phoff: 64,
+            shoff,
+            flags: 0,
+            ehsize: 64,
+            phentsize: 56,
+            phnum: 2,
+            shentsize: 64,
+            shnum,
+            shstrndx,
+        }
+    }
+
+    fn create_program_headers(&self, output_sections: &[Section<'static>]) -> Vec<ProgramHeader> {
+        let mut program_headers = Vec::new();
+
+        for section in output_sections.iter() {
+            match section.name.as_ref() {
+                ".text" => {
+                    let text_ph = ProgramHeader {
+                        r#type: SegmentType::Load,
+                        flags: vec![Flag::Readable, Flag::Executable],
+                        offset: 0,
+                        vaddr: BASE_ADDR,
+                        paddr: BASE_ADDR,
+                        filesz: section.offset + section.size,
+                        memsz: section.offset + section.size,
+                        align: 0x10000,
+                    };
+                    program_headers.push(text_ph);
+                }
+                ".data" => {
+                    let data_ph = ProgramHeader {
+                        r#type: SegmentType::Load,
+                        flags: vec![Flag::Readable, Flag::Writable],
+                        offset: section.offset,
+                        vaddr: section.addr,
+                        paddr: section.addr,
+                        filesz: section.size,
+                        memsz: section.size,
+                        align: 0x10000,
+                    };
+                    program_headers.push(data_ph);
+                }
+                _ => {}
+            }
+        }
+
+        program_headers
+    }
+
+    fn write_program_headers<W: Write>(
+        &self,
+        writer: &mut W,
+        headers: &[ProgramHeader],
+    ) -> Result<()> {
+        for ph in headers {
+            let mut flag: u32 = 0;
+            for f in &ph.flags {
+                flag |= *f as u32;
+            }
+
+            let bytes = [
+                (ph.r#type as u32).to_le_bytes().as_slice(),
+                flag.to_le_bytes().as_slice(),
+                ph.offset.to_le_bytes().as_slice(),
+                ph.vaddr.to_le_bytes().as_slice(),
+                ph.paddr.to_le_bytes().as_slice(),
+                ph.filesz.to_le_bytes().as_slice(),
+                ph.memsz.to_le_bytes().as_slice(),
+                ph.align.to_le_bytes().as_slice(),
+            ]
+            .concat();
+
+            writer.write_all(&bytes)?;
+        }
+
+        Ok(())
+    }
+
+    fn write_sections<W: Write + Seek>(
+        &self,
+        writer: &mut W,
+        sections: &[Section<'static>],
+    ) -> Result<()> {
+        for section in sections {
+            writer.seek(SeekFrom::Start(section.offset))?;
+            writer.write_all(section.data.as_ref())?;
+        }
+
+        Ok(())
+    }
+
+    fn write_section_header<W: Write>(
+        &self,
+        writer: &mut W,
+        name: u32,
+        sh_type: u32,
+        sh_flags: u64,
+        sh_addr: u64,
+        sh_offset: u64,
+        sh_size: u64,
+        sh_link: u32,
+        sh_info: u32,
+        sh_addralign: u64,
+        sh_entsize: u64,
+    ) -> Result<()> {
+        let bytes = [
+            name.to_le_bytes().as_slice(),
+            sh_type.to_le_bytes().as_slice(),
+            sh_flags.to_le_bytes().as_slice(),
+            sh_addr.to_le_bytes().as_slice(),
+            sh_offset.to_le_bytes().as_slice(),
+            sh_size.to_le_bytes().as_slice(),
+            sh_link.to_le_bytes().as_slice(),
+            sh_info.to_le_bytes().as_slice(),
+            sh_addralign.to_le_bytes().as_slice(),
+            sh_entsize.to_le_bytes().as_slice(),
+        ]
+        .concat();
+
+        writer.write_all(&bytes)?;
+        Ok(())
     }
 }
```

### テストを実行する

```sh
$ cargo test linker::writer
running 1 test
test linker::writer::tests::write_executable_writes_valid_elf_header ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## 出力ファイルの構造

最終的にでき上がるELFファイルのレイアウトは次のようになる。

```
ファイルオフセット
┌─────────────────┬──────────────┐
│ 0x0000          │ ELF Header   │  64バイト
├─────────────────┼──────────────┤
│ 0x0040          │ Program      │  56バイト × 2
│                 │ Header       │
├─────────────────┼──────────────┤
│ 0x0100          │ .text        │  コードデータ
├─────────────────┼──────────────┤
│ 0x0110          │ .data        │  データ
├─────────────────┼──────────────┤
│ 0x0114          │ .strtab      │  シンボル名の文字列テーブル
├─────────────────┼──────────────┤
│ 0x0138          │ .symtab      │  シンボルテーブル
├─────────────────┼──────────────┤
│ 0x01b0          │ .shstrtab    │  セクション名の文字列テーブル
├─────────────────┼──────────────┤
│ 0x01d8          │ Section      │  64バイト × 6
│                 │ Header       │
└─────────────────┴──────────────┘
```

## まとめ

本章ではELFファイルへの書き出しを実装した。

- ELFヘッダー：ファイルの種類、エントリポイント、セクションヘッダーの位置などを記録する
- プログラムヘッダー：OSがメモリにロードするセグメントの情報
- セクションヘッダー：各セクションのメタデータ

これでリンカーの全パーツがそろった。次章ではすべてを統合し、`tiny-linker`コマンドとして動かせるようにする。
