---
title: "セクション配置の実装"
---

本章では、セクション配置の実装を行う。複数のオブジェクトファイルからセクションをマージし、メモリアドレスを割り当てる。

## 出力セクションの設計

出力セクションのデータ構造を定義する。

```rust:src/linker/output.rs
use std::borrow::Cow;
use crate::elf::section::{SectionFlag, SectionType};

#[derive(Debug, Clone)]
pub struct Section<'a> {
    pub name: Cow<'a, str>,
    pub r#type: SectionType,
    pub flags: Vec<SectionFlag>,
    pub addr: u64,      // 仮想アドレス
    pub offset: u64,    // ファイル内オフセット
    pub size: u64,
    pub data: Cow<'a, [u8]>,
    pub align: u64,
}
```

## アラインメント関数

アドレスやオフセットを指定した境界に揃えるヘルパー関数を実装する。

```rust:src/linker/section.rs
/// 値を指定した2のべき乗の境界に揃える
pub fn align(value: u64, alignment: u64) -> u64 {
    (value + alignment - 1) & !(alignment - 1)
}
```

例えば、`align(10, 8)`は`16`を返す（10を8の倍数に切り上げ）。

## セクションのマージ

複数のオブジェクトファイルから`.text`と`.data`セクションをマージする処理を実装する。

```rust:src/linker/section.rs
use std::borrow::Cow;
use std::collections::HashMap;
use crate::elf::section::{SectionFlag, SectionType};
use crate::error::Result;
use super::Linker;
use super::output::{ResolvedSymbol, Section};

/// 実行可能ファイルのベースアドレス
pub static BASE_ADDR: u64 = 0x400000;

impl Linker {
    fn merge_sections(
        &self,
        resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
        base_addr: u64,
    ) -> Result<Vec<Section<'static>>> {
        // マージ後のセクションデータ
        let mut raw_text_section = vec![];
        let mut raw_data_section = vec![];

        // 各オブジェクトのセクションがマージ後のどこに配置されるかを記録
        // キー: (オブジェクトインデックス, セクションインデックス)
        // 値: マージ後のオフセット
        let mut text_offsets = HashMap::new();
        let mut data_offsets = HashMap::new();

        let mut text_current_offset = 0;
        let mut data_current_offset = 0;

        // 各オブジェクトファイルのセクションを処理
        for (obj_idx, obj) in self.objects.iter().enumerate() {
            for (section_idx, section) in obj.section_headers.iter().enumerate() {
                match section.name.as_str() {
                    ".text" => {
                        // オフセットを記録
                        text_offsets.insert(
                            (obj_idx, section_idx as u16),
                            text_current_offset
                        );
                        // データを結合
                        raw_text_section.extend_from_slice(&section.section_raw_data);
                        text_current_offset += section.section_raw_data.len();
                    }
                    ".data" => {
                        data_offsets.insert(
                            (obj_idx, section_idx as u16),
                            data_current_offset
                        );
                        raw_data_section.extend_from_slice(&section.section_raw_data);
                        data_current_offset += section.section_raw_data.len();
                    }
                    _ => {
                        // 他のセクションは無視
                    }
                }
            }
        }

        // .textセクションの配置
        // ELFヘッダー(64バイト) + プログラムヘッダー(56バイト×2) の後
        let text_offset = 0x100;
        let text_addr = align(base_addr + text_offset, 4);

        let text_section = Section {
            name: Cow::Borrowed(".text"),
            r#type: SectionType::ProgBits,
            flags: vec![SectionFlag::Alloc, SectionFlag::ExecInstr],
            addr: text_addr,
            offset: text_offset,
            size: raw_text_section.len() as u64,
            data: Cow::Owned(raw_text_section),
            align: 4,
        };

        // .dataセクションの配置
        // .textの直後に配置
        let data_offset = text_offset + text_section.size;
        // メモリ上は0x10000のギャップを設ける（ページ境界）
        let data_base_addr = align(text_section.addr + text_section.size + 0x10000, 4);

        let data_section = Section {
            name: Cow::Borrowed(".data"),
            r#type: SectionType::ProgBits,
            flags: vec![SectionFlag::Alloc, SectionFlag::Write],
            addr: data_base_addr,
            offset: data_offset,
            size: raw_data_section.len() as u64,
            data: Cow::Owned(raw_data_section),
            align: 4,
        };

        // シンボルのアドレスを更新
        for symbol in resolved_symbols.values_mut() {
            if let Some(&offset) = text_offsets.get(&(symbol.object_index, symbol.shndx)) {
                // .textセクション内のシンボル
                // 新しいアドレス = セクションアドレス + マージ後のオフセット + 元のオフセット
                symbol.value = text_section.addr + (offset + symbol.value as usize) as u64;
            } else if let Some(&offset) = data_offsets.get(&(symbol.object_index, symbol.shndx)) {
                // .dataセクション内のシンボル
                symbol.value = data_section.addr + (offset + symbol.value as usize) as u64;
            }
        }

        let mut output_sections = vec![text_section, data_section];

        // 再配置を適用（次章で実装）
        self.apply_relocations(&mut output_sections, resolved_symbols)?;

        Ok(output_sections)
    }
}
```

## シンボルアドレスの更新

セクションをマージした後、各シンボルのアドレスを更新する必要がある。

例えば、`sub.o`の変数`x`は元々`.data`セクションのオフセット0にあった。
マージ後は次のようにアドレスが計算される。

```
x のアドレス = .dataセクションのアドレス + sub.oの.dataオフセット + 元のオフセット
            = 0x410110 + 0 + 0
            = 0x410110
```

同様に、`main.o`の`_start`は`.text`セクションのオフセット0にあった。

```
_start のアドレス = .textセクションのアドレス + main.oの.textオフセット + 元のオフセット
                 = 0x400100 + 0 + 0
                 = 0x400100
```

## layout_sections関数

セクション配置の全体を管理する関数を実装する。

```rust:src/linker/section.rs
impl Linker {
    pub fn layout_sections(
        &self,
        resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
    ) -> Result<(Vec<Section<'static>>, HashMap<String, usize>)> {
        // セクションのマージと再配置
        let output_sections = self.merge_sections(resolved_symbols, BASE_ADDR)?;

        // 文字列テーブルとシンボルテーブルの作成
        let latest_offset = output_sections.last().unwrap().offset
            + output_sections.last().unwrap().size;
        let (symtab_section, strtab_section) =
            self.make_symbol_section(latest_offset, resolved_symbols);

        // セクションヘッダー文字列テーブルの作成
        let mut shstrtab: Vec<u8> = vec![0]; // NULL文字から始める
        let mut section_name_offsets: HashMap<String, usize> = HashMap::new();

        // 各セクション名を追加
        for section in output_sections.iter() {
            section_name_offsets.insert(section.name.to_string(), shstrtab.len());
            shstrtab.extend_from_slice(section.name.as_bytes());
            shstrtab.push(0);
        }

        // .strtab, .symtab, .shstrtab の名前を追加
        for name in [".strtab", ".symtab", ".shstrtab"] {
            section_name_offsets.insert(name.to_string(), shstrtab.len());
            shstrtab.extend_from_slice(name.as_bytes());
            shstrtab.push(0);
        }

        let shstrtab_section = Section {
            name: Cow::Borrowed(".shstrtab"),
            r#type: SectionType::StrTab,
            flags: vec![],
            addr: 0,
            offset: align(symtab_section.offset + symtab_section.size, 8),
            size: shstrtab.len() as u64,
            data: Cow::Owned(shstrtab),
            align: 1,
        };

        // すべてのセクションを結合
        let mut all_sections = output_sections;
        all_sections.push(strtab_section);
        all_sections.push(symtab_section);
        all_sections.push(shstrtab_section);

        Ok((all_sections, section_name_offsets))
    }
}
```

これでセクション配置の実装が完了した。次章では、再配置の適用を実装する。
