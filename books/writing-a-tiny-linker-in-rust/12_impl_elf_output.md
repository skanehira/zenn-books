---
title: "ELF出力の実装"
---

本章では、実行可能なELFファイルを出力する処理を実装する。

## 本章で実装するファイル

```
src/
├── elf/
│   ├── header.rs         # 本章（to_vec追加）
│   ├── program_header.rs # 本章
│   └── segment.rs        # 本章
├── linker/
│   ├── mod.rs            # 本章（モジュール追加）
│   └── writer.rs         # 本章
└── ...
```

## モジュール構造を更新する

LSPが正しく動作するように、最初にモジュール宣言を追加する。

### elf.rsを更新する

```diff:src/elf.rs
 pub mod header;
+pub mod program_header;
 pub mod relocation;
 pub mod section;
+pub mod segment;
 pub mod symbol;
```

### linker/mod.rsを更新する

```diff:src/linker/mod.rs
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

## プログラムヘッダーのデータ構造

### テストを書く

`src/elf/segment.rs`にテストを書く。

```rust:src/elf/segment.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn flag_values() {
        assert_eq!(Flag::Executable as u32, 0x1);
        assert_eq!(Flag::Writable as u32, 0x2);
        assert_eq!(Flag::Readable as u32, 0x4);
    }
}
```

### セグメント型を実装する

```diff:src/elf/segment.rs
+#[derive(Debug, Clone, Copy)]
+#[repr(u32)]
+pub enum Type {
+    Load = 1,
+}
+
+#[derive(Debug, Clone, Copy)]
+#[repr(u32)]
+pub enum Flag {
+    Executable = 0x1,
+    Writable = 0x2,
+    Readable = 0x4,
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test elf::segment::tests::flag_values
running 1 test
test elf::segment::tests::flag_values ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

### テストを書く

`src/elf/program_header.rs`にテストを書く。

```rust:src/elf/program_header.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::elf::segment::{Flag, Type};

    #[test]
    fn program_header_creation() {
        let ph = ProgramHeader {
            r#type: Type::Load,
            flags: vec![Flag::Readable, Flag::Executable],
            offset: 0,
            vaddr: 0x400000,
            paddr: 0x400000,
            filesz: 0x100,
            memsz: 0x100,
            align: 0x10000,
        };

        assert_eq!(ph.vaddr, 0x400000);
        assert_eq!(ph.flags.len(), 2);
    }
}
```

### ProgramHeader構造体を実装する

```diff:src/elf/program_header.rs
+use super::segment::{Flag, Type};
+
+#[derive(Debug, Clone)]
+pub struct ProgramHeader {
+    pub r#type: Type,
+    pub flags: Vec<Flag>,
+    pub offset: u64,
+    pub vaddr: u64,
+    pub paddr: u64,
+    pub filesz: u64,
+    pub memsz: u64,
+    pub align: u64,
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test elf::program_header::tests::program_header_creation
running 1 test
test elf::program_header::tests::program_header_creation ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## ELFヘッダーのシリアライズ

### テストを書く

`src/elf/header.rs`にシリアライズのテストを追加する。

```diff:src/elf/header.rs
 #[cfg(test)]
 mod tests {
+    use super::*;
+
+    #[test]
+    fn header_to_vec() {
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
+        let bytes = header.to_vec();
+        assert_eq!(bytes.len(), 64);
+
+        // マジックナンバー
+        assert_eq!(&bytes[0..4], &[0x7f, b'E', b'L', b'F']);
+    }
 }
```

### to_vecメソッドを実装する

```diff:src/elf/header.rs
 impl Header {
+    pub fn to_vec(&self) -> Vec<u8> {
+        let mut bytes = Vec::with_capacity(64);
+
+        // e_ident (16バイト)
+        bytes.extend_from_slice(&[0x7f, b'E', b'L', b'F']); // マジック
+        bytes.push(self.ident.class as u8);
+        bytes.push(self.ident.data as u8);
+        bytes.push(self.ident.version as u8);
+        bytes.push(self.ident.os_abi as u8);
+        bytes.push(self.ident.abi_version);
+        bytes.extend_from_slice(&[0u8; 7]); // パディング
+
+        // e_type (2バイト)
+        bytes.extend_from_slice(&(self.r#type as u16).to_le_bytes());
+        // e_machine (2バイト)
+        bytes.extend_from_slice(&(self.machine as u16).to_le_bytes());
+        // e_version (4バイト)
+        bytes.extend_from_slice(&(self.version as u32).to_le_bytes());
+        // e_entry (8バイト)
+        bytes.extend_from_slice(&self.entry.to_le_bytes());
+        // e_phoff (8バイト)
+        bytes.extend_from_slice(&self.phoff.to_le_bytes());
+        // e_shoff (8バイト)
+        bytes.extend_from_slice(&self.shoff.to_le_bytes());
+        // e_flags (4バイト)
+        bytes.extend_from_slice(&self.flags.to_le_bytes());
+        // e_ehsize (2バイト)
+        bytes.extend_from_slice(&self.ehsize.to_le_bytes());
+        // e_phentsize (2バイト)
+        bytes.extend_from_slice(&self.phentsize.to_le_bytes());
+        // e_phnum (2バイト)
+        bytes.extend_from_slice(&self.phnum.to_le_bytes());
+        // e_shentsize (2バイト)
+        bytes.extend_from_slice(&self.shentsize.to_le_bytes());
+        // e_shnum (2バイト)
+        bytes.extend_from_slice(&self.shnum.to_le_bytes());
+        // e_shstrndx (2バイト)
+        bytes.extend_from_slice(&self.shstrndx.to_le_bytes());
+
+        bytes
+    }
 }

 #[cfg(test)]
```

### テストを実行する

```sh
$ cargo test elf::header::tests::header_to_vec
running 1 test
test elf::header::tests::header_to_vec ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## ELF出力の実装

### テストを書く

`src/linker/writer.rs`にテストを書く。

```rust:src/linker/writer.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::linker::Linker;
    use crate::parser::Elf;
    use std::io::Cursor;

    #[test]
    fn write_executable() {
        let main_o = include_bytes!("../parser/fixtures/main.o");
        let sub_o = include_bytes!("../parser/fixtures/sub.o");

        let mut linker = Linker::new();
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

```diff:src/linker/writer.rs
+use std::collections::HashMap;
+use std::io::{Seek, SeekFrom, Write};
+
+use crate::elf::header::{self, Class, Data, Ident, IdentVersion, Machine, OSABI, Type, Version};
+use crate::elf::program_header::ProgramHeader;
+use crate::elf::segment::{Flag, Type as SegmentType};
+use crate::elf::symbol::Binding;
+use crate::error::{LinkerError, Result};
+
+use super::output::{ResolvedSymbol, Section};
+use super::section::{align, BASE_ADDR};
+use super::Linker;
+
+impl Linker {
+    pub fn write_executable<W: Write + Seek>(
+        &self,
+        writer: &mut W,
+        resolved_symbols: HashMap<String, ResolvedSymbol>,
+        section_tables: Vec<Section<'static>>,
+        section_name_offsets: HashMap<String, usize>,
+    ) -> Result<()> {
+        // エントリポイント（_start）のアドレスを取得
+        let entry = resolved_symbols
+            .get("_start")
+            .ok_or(LinkerError::MissingEntryPoint)?
+            .value;
+
+        // ELFヘッダーを書き出し
+        let elf_header = self.create_elf_header(entry, &section_tables);
+        writer.write_all(&elf_header.to_vec())?;
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
+    }
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test linker::writer
running 1 test
test linker::writer::tests::write_executable ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
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

## まとめ

本章ではELF出力を実装した。

- モジュール宣言を先に追加してLSPを有効化
- ELFヘッダー: ファイルの種類、エントリポイント、セクションヘッダーの位置などを記録
- プログラムヘッダー: OSがメモリにロードするセグメント情報
- セクションヘッダー: 各セクションのメタデータ

次章では、すべてを統合して実行可能バイナリを生成する。
