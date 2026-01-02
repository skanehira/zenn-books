---
title: "ELF出力の実装"
---

本章では、実行可能なELFファイルを出力する処理を実装する。

## ELFヘッダーの生成

実行可能ファイル用のELFヘッダーを生成する。

```rust:src/linker/writer.rs
use crate::elf::header::{self, Class, Data, Ident, IdentVersion, Machine, OSABI, Type, Version};
use super::output::Section;
use super::section::{align, BASE_ADDR};

impl Linker {
    fn create_elf_header(
        &self,
        entry: u64,
        section_tables: &[Section<'static>]
    ) -> header::Header {
        // セクションヘッダーのオフセットは全セクションの後
        let shoff = section_tables
            .iter()
            .map(|s| s.offset + align(s.size, 8))
            .max()
            .unwrap();

        // セクション数（NULLセクションを含む）
        let shnum = (section_tables.len() + 1) as u16;

        // .shstrtabのインデックス
        let shstrndx = section_tables
            .iter()
            .position(|s| s.name == ".shstrtab")
            .map(|i| (i + 1) as u16)  // NULLセクションの分を加算
            .unwrap_or(0);

        header::Header {
            ident: Ident {
                class: Class::Bit64,
                data: Data::Lsb,
                version: IdentVersion::Current,
                os_abi: OSABI::SystemV,
                abi_version: 0,
            },
            r#type: Type::Exec,          // 実行可能ファイル
            machine: Machine::AArch64,   // ARM64
            version: Version::Current,
            entry,                        // エントリポイント（_startのアドレス）
            phoff: 64,                    // プログラムヘッダーはELFヘッダーの直後
            shoff,
            flags: 0,
            ehsize: 64,                   // ELFヘッダーサイズ
            phentsize: 56,                // プログラムヘッダーエントリサイズ
            phnum: 2,                     // プログラムヘッダー数（.text + .data）
            shentsize: 64,                // セクションヘッダーエントリサイズ
            shnum,
            shstrndx,
        }
    }
}
```

## プログラムヘッダーの生成

OSがメモリにロードするセグメントを定義するプログラムヘッダーを生成する。

```rust:src/elf/program_header.rs
use super::segument::{Flag, Type};

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

```rust:src/elf/segument.rs
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

```rust:src/linker/writer.rs
impl Linker {
    fn create_program_headers(
        &self,
        output_sections: &[Section<'static>],
    ) -> Vec<ProgramHeader> {
        let mut program_headers = Vec::new();

        for section in output_sections.iter() {
            match section.name.as_ref() {
                ".text" => {
                    // コードセグメント（読み取り + 実行）
                    let text_ph = ProgramHeader {
                        r#type: Type::Load,
                        flags: vec![Flag::Readable, Flag::Executable],
                        offset: 0,
                        vaddr: BASE_ADDR,
                        paddr: BASE_ADDR,
                        filesz: section.size,
                        memsz: section.size,
                        align: 0x10000,
                    };
                    program_headers.push(text_ph);
                }
                ".data" => {
                    // データセグメント（読み取り + 書き込み）
                    let data_ph = ProgramHeader {
                        r#type: Type::Load,
                        flags: vec![Flag::Readable, Flag::Writable],
                        offset: section.offset,
                        vaddr: section.addr,
                        paddr: section.addr,
                        filesz: section.size,
                        memsz: section.size,
                        align: 0x10000,
                    };
                    program_headers.push(data_ph);
                }
                _ => {}
            }
        }

        program_headers
    }
}
```

## セクションヘッダーの生成

各セクションのメタデータを格納するセクションヘッダーを書き出す。

```rust:src/linker/writer.rs
use std::io::{Seek, SeekFrom, Write};
use std::collections::HashMap;
use crate::error::{LinkerError, Result};
use super::output::ResolvedSymbol;

impl Linker {
    pub fn write_executable<W: Write + Seek>(
        &self,
        writer: &mut W,
        resolved_symbols: HashMap<String, ResolvedSymbol>,
        section_tables: Vec<Section<'static>>,
        section_name_offsets: HashMap<String, usize>,
    ) -> Result<()> {
        // エントリポイント（_start）のアドレスを取得
        let entry = resolved_symbols
            .get("_start")
            .ok_or(LinkerError::MissingEntryPoint)?
            .value;

        // ELFヘッダーを書き出し
        let elf_header = self.create_elf_header(entry, &section_tables);
        writer.write_all(&elf_header.to_vec())?;

        // プログラムヘッダーを書き出し
        let program_headers = self.create_program_headers(&section_tables);
        self.write_program_headers(writer, &program_headers)?;

        // セクションデータを書き出し
        self.write_sections(writer, &section_tables)?;

        // セクションヘッダーテーブルの位置に移動
        writer.seek(SeekFrom::Start(elf_header.shoff))?;

        // NULLセクションヘッダー
        self.write_section_header(writer, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)?;

        // 各セクションのヘッダー
        for section in section_tables.iter() {
            let name_offset = *section_name_offsets
                .get(section.name.as_ref())
                .unwrap_or(&0);

            let mut sh_flags: u64 = 0;
            for f in &section.flags {
                sh_flags |= *f as u64;
            }

            // .symtabの場合は特別な処理
            let (sh_link, sh_info) = if section.name == ".symtab" {
                // sh_link: .strtabのインデックス
                let strtab_idx = section_tables
                    .iter()
                    .position(|s| s.name == ".strtab")
                    .map(|i| i + 1)  // NULLセクションを含む
                    .unwrap_or(0) as u32;

                // sh_info: 最初のグローバルシンボルのインデックス
                let local_count = resolved_symbols
                    .values()
                    .filter(|s| s.info.binding == Binding::Local)
                    .count() as u32 + 1;  // NULLシンボルを含む

                (strtab_idx, local_count)
            } else {
                (0, 0)
            };

            let sh_entsize = if section.name == ".symtab" { 24 } else { 0 };

            self.write_section_header(
                writer,
                name_offset as u32,
                section.r#type as u32,
                sh_flags,
                section.addr,
                section.offset,
                section.size,
                sh_link,
                sh_info,
                section.align,
                sh_entsize,
            )?;
        }

        Ok(())
    }

    fn write_section_header<W: Write>(
        &self,
        writer: &mut W,
        name: u32,
        sh_type: u32,
        sh_flags: u64,
        sh_addr: u64,
        sh_offset: u64,
        sh_size: u64,
        sh_link: u32,
        sh_info: u32,
        sh_addralign: u64,
        sh_entsize: u64,
    ) -> Result<()> {
        let bytes = [
            name.to_le_bytes().as_slice(),
            sh_type.to_le_bytes().as_slice(),
            sh_flags.to_le_bytes().as_slice(),
            sh_addr.to_le_bytes().as_slice(),
            sh_offset.to_le_bytes().as_slice(),
            sh_size.to_le_bytes().as_slice(),
            sh_link.to_le_bytes().as_slice(),
            sh_info.to_le_bytes().as_slice(),
            sh_addralign.to_le_bytes().as_slice(),
            sh_entsize.to_le_bytes().as_slice(),
        ]
        .concat();

        writer.write_all(&bytes)?;
        Ok(())
    }
}
```

## プログラムヘッダーの書き出し

```rust:src/linker/writer.rs
impl Linker {
    fn write_program_headers<W: Write>(
        &self,
        writer: &mut W,
        headers: &[ProgramHeader],
    ) -> Result<()> {
        for ph in headers {
            let mut flag: u32 = 0;
            for f in &ph.flags {
                flag |= *f as u32;
            }

            let bytes = [
                (ph.r#type as u32).to_le_bytes().as_slice(),
                flag.to_le_bytes().as_slice(),
                ph.offset.to_le_bytes().as_slice(),
                ph.vaddr.to_le_bytes().as_slice(),
                ph.paddr.to_le_bytes().as_slice(),
                ph.filesz.to_le_bytes().as_slice(),
                ph.memsz.to_le_bytes().as_slice(),
                ph.align.to_le_bytes().as_slice(),
            ]
            .concat();

            writer.write_all(&bytes)?;
        }

        Ok(())
    }
}
```

## セクションデータの書き出し

```rust:src/linker/writer.rs
impl Linker {
    fn write_sections<W: Write + Seek>(
        &self,
        writer: &mut W,
        sections: &[Section<'static>],
    ) -> Result<()> {
        for section in sections {
            // セクションのオフセット位置に移動
            writer.seek(SeekFrom::Start(section.offset))?;
            // セクションデータを書き出し
            writer.write_all(section.data.as_ref())?;
        }

        Ok(())
    }
}
```

## 出力ファイルの構造

最終的に出力されるELFファイルの構造は次のようになる。

```
ファイルオフセット
┌─────────────────┬──────────────┐
│ 0x0000          │ ELFヘッダー  │  64バイト
├─────────────────┼──────────────┤
│ 0x0040          │ プログラム   │  56バイト × 2
│                 │ ヘッダー     │
├─────────────────┼──────────────┤
│ 0x0100          │ .text        │  コードデータ
├─────────────────┼──────────────┤
│ 0x0110          │ .data        │  データ
├─────────────────┼──────────────┤
│ ...             │ .strtab      │  文字列テーブル
├─────────────────┼──────────────┤
│ ...             │ .symtab      │  シンボルテーブル
├─────────────────┼──────────────┤
│ ...             │ .shstrtab    │  セクション名テーブル
├─────────────────┼──────────────┤
│ 0x01e0          │ セクション   │  64バイト × 6
│                 │ ヘッダー     │
└─────────────────┴──────────────┘
```

これでELF出力の実装が完了した。次章では、すべてを統合して実行可能バイナリを生成する。
