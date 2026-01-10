---
title: "セクション配置の実装"
---

本章では、セクション配置の実装を行う。複数のオブジェクトファイルからセクションをマージし、メモリアドレスを割り当てる。

## 本章で実装するファイル

```
src/
├── linker.rs          # 本章（モジュール追加）
├── linker/
│   ├── output.rs      # 本章（Section追加）
│   └── section.rs     # 本章
└── ...
```

## モジュール構造を更新する

### linker.rsを更新する

```diff:src/linker.rs
 pub mod output;
+pub mod section;
 pub mod symbol;

 use crate::parser::Elf;
```

### 空のモジュールファイルを作成する

```sh
$ touch src/linker/section.rs
```

## 出力セクションの構造体

### テストを書く

`src/linker/output.rs`にSection構造体とテストを追加する。

```diff:src/linker/output.rs
+use std::borrow::Cow;
+
+use crate::elf::section::{SectionFlag, SectionType};
 use crate::elf::symbol::{self, Binding};

+#[derive(Debug, Clone)]
+pub struct Section<'a> {
+    pub name: Cow<'a, str>,
+    pub r#type: SectionType,
+    pub flags: Vec<SectionFlag>,
+    pub addr: u64,
+    pub offset: u64,
+    pub size: u64,
+    pub data: Cow<'a, [u8]>,
+    pub align: u64,
+}
+
 #[derive(Debug, Clone)]
 pub struct ResolvedSymbol {
```

```diff:src/linker/output.rs
 #[cfg(test)]
 mod tests {
     use super::*;
-    use crate::elf::symbol::SymbolType;
+    use crate::elf::symbol::{Binding, SymbolType};
+
+    #[test]
+    fn section_creation() {
+        let section = Section {
+            name: Cow::Borrowed(".text"),
+            r#type: SectionType::ProgBits,
+            flags: vec![SectionFlag::Alloc, SectionFlag::ExecInstr],
+            addr: 0x400100,
+            offset: 0x100,
+            size: 16,
+            data: Cow::Owned(vec![0u8; 16]),
+            align: 4,
+        };
+
+        assert_eq!(section.name, ".text");
+        assert_eq!(section.addr, 0x400100);
+    }

     fn make_symbol(binding: Binding, is_defined: bool) -> ResolvedSymbol {
```

### テストを実行する

```sh
$ cargo test linker::output::tests::section_creation
running 1 test
test linker::output::tests::section_creation ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## アラインメント関数

### テストを書く

`src/linker/section.rs`にテストを書く。テストをコンパイルするために、最小限のスタブも一緒に追加する。

```rust:src/linker/section.rs
/// 値を指定した2のべき乗の境界に揃える
pub fn align(_value: u64, _alignment: u64) -> u64 {
    todo!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn align_returns_aligned_value_when_value_not_aligned() {
        // 8の倍数に切り上げ
        assert_eq!(align(0, 8), 0);
        assert_eq!(align(1, 8), 8);
        assert_eq!(align(7, 8), 8);
        assert_eq!(align(8, 8), 8);
        assert_eq!(align(9, 8), 16);

        // 4の倍数に切り上げ
        assert_eq!(align(1, 4), 4);
        assert_eq!(align(4, 4), 4);
        assert_eq!(align(5, 4), 8);
    }
}
```

### align関数を実装する

```diff:src/linker/section.rs
 /// 値を指定した2のべき乗の境界に揃える
-pub fn align(_value: u64, _alignment: u64) -> u64 {
-    todo!()
+pub fn align(value: u64, alignment: u64) -> u64 {
+    (value + alignment - 1) & !(alignment - 1)
 }
```

### テストを実行する

```sh
$ cargo test linker::section::tests::align_returns_aligned_value
running 1 test
test linker::section::tests::align_returns_aligned_value_when_value_not_aligned ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## セクション配置の実装

### テストを書く

`src/linker/section.rs`にテストを書く。テストをコンパイルするために、最小限のスタブも一緒に追加する。

```diff:src/linker/section.rs
+use std::borrow::Cow;
+use std::collections::HashMap;
+
+use crate::elf::section::{SectionFlag, SectionType};
+use crate::error::Result;
+
+use super::output::{ResolvedSymbol, Section};
+use super::Linker;
+
+/// 実行可能ファイルのベースアドレス
+pub static BASE_ADDR: u64 = 0x400000;
+
 /// 値を指定した2のべき乗の境界に揃える
 pub fn align(value: u64, alignment: u64) -> u64 {
     (value + alignment - 1) & !(alignment - 1)
 }
+
+impl Linker {
+    pub fn layout_sections(
+        &self,
+        _resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
+    ) -> Result<(Vec<Section<'static>>, HashMap<String, usize>)> {
+        todo!()
+    }
+}
```

```diff:src/linker/section.rs
 #[cfg(test)]
 mod tests {
     use super::*;
+    use crate::linker::Linker;
+    use crate::parser::Elf;

     #[test]
     fn align_returns_aligned_value_when_value_not_aligned() {
         // 8の倍数に切り上げ
         assert_eq!(align(0, 8), 0);
         assert_eq!(align(1, 8), 8);
         assert_eq!(align(7, 8), 8);
         assert_eq!(align(8, 8), 8);
         assert_eq!(align(9, 8), 16);

         // 4の倍数に切り上げ
         assert_eq!(align(1, 4), 4);
         assert_eq!(align(4, 4), 4);
         assert_eq!(align(5, 4), 8);
     }
+
+    #[test]
+    fn layout_sections_returns_sections_with_correct_addresses() {
+        let main_o = include_bytes!("../parser/fixtures/main.o");
+        let sub_o = include_bytes!("../parser/fixtures/sub.o");
+
+        let mut linker = Linker::new();
+        linker.add_object("main.o".to_string(), Elf::parse(main_o).unwrap());
+        linker.add_object("sub.o".to_string(), Elf::parse(sub_o).unwrap());
+
+        let mut resolved = linker.resolve_symbols().unwrap();
+        let (sections, _) = linker.layout_sections(&mut resolved).unwrap();
+
+        // .textセクションが正しく配置されている
+        let text = sections.iter().find(|s| s.name == ".text").unwrap();
+        assert_eq!(text.addr, 0x400100);
+
+        // .dataセクションが正しく配置されている
+        let data = sections.iter().find(|s| s.name == ".data").unwrap();
+        // .textの後に0x10000のギャップを設けて配置
+        assert!(data.addr > text.addr + text.size);
+
+        // シンボルのアドレスが更新されている
+        let start = resolved.get("_start").unwrap();
+        assert_eq!(start.value, 0x400100);
+
+        let x = resolved.get("x").unwrap();
+        assert!(x.value >= data.addr);
+    }
 }
```

### セクション配置を実装する

```diff:src/linker/section.rs
 impl Linker {
     pub fn layout_sections(
         &self,
-        _resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
+        resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
     ) -> Result<(Vec<Section<'static>>, HashMap<String, usize>)> {
-        todo!()
+        // セクションのマージ
+        let output_sections = self.merge_sections(resolved_symbols, BASE_ADDR)?;
+
+        // シンボルテーブルと文字列テーブルの作成
+        let latest_offset =
+            output_sections.last().unwrap().offset + output_sections.last().unwrap().size;
+        let (symtab_section, strtab_section) =
+            self.make_symbol_section(latest_offset, resolved_symbols);
+
+        // セクションヘッダー文字列テーブルの作成
+        let mut shstrtab: Vec<u8> = vec![0]; // NULL文字から始める
+        let mut section_name_offsets: HashMap<String, usize> = HashMap::new();
+
+        // 各セクション名を追加
+        for section in output_sections.iter() {
+            section_name_offsets.insert(section.name.to_string(), shstrtab.len());
+            shstrtab.extend_from_slice(section.name.as_bytes());
+            shstrtab.push(0);
+        }
+
+        // .strtab, .symtab, .shstrtab の名前を追加
+        for name in [".strtab", ".symtab", ".shstrtab"] {
+            section_name_offsets.insert(name.to_string(), shstrtab.len());
+            shstrtab.extend_from_slice(name.as_bytes());
+            shstrtab.push(0);
+        }
+
+        let shstrtab_section = Section {
+            name: Cow::Borrowed(".shstrtab"),
+            r#type: SectionType::StrTab,
+            flags: vec![],
+            addr: 0,
+            offset: align(symtab_section.offset + symtab_section.size, 8),
+            size: shstrtab.len() as u64,
+            data: Cow::Owned(shstrtab),
+            align: 1,
+        };
+
+        // すべてのセクションを結合
+        let mut all_sections = output_sections;
+        all_sections.push(strtab_section);
+        all_sections.push(symtab_section);
+        all_sections.push(shstrtab_section);
+
+        Ok((all_sections, section_name_offsets))
+    }
+
+    fn merge_sections(
+        &self,
+        resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
+        base_addr: u64,
+    ) -> Result<Vec<Section<'static>>> {
+        // マージ後のセクションデータ
+        let mut raw_text_section = vec![];
+        let mut raw_data_section = vec![];
+
+        // 各オブジェクトのセクションがマージ後のどこに配置されるかを記録
+        // キー: (オブジェクトインデックス, セクションインデックス)
+        // 値: マージ後のオフセット
+        let mut text_offsets: HashMap<(usize, u16), usize> = HashMap::new();
+        let mut data_offsets: HashMap<(usize, u16), usize> = HashMap::new();
+
+        let mut text_current_offset = 0;
+        let mut data_current_offset = 0;
+
+        // 各オブジェクトファイルのセクションを処理
+        for (obj_idx, obj) in self.objects.iter().enumerate() {
+            for (section_idx, section) in obj.section_headers.iter().enumerate() {
+                match section.name.as_str() {
+                    ".text" => {
+                        text_offsets.insert((obj_idx, section_idx as u16), text_current_offset);
+                        raw_text_section.extend_from_slice(&section.data);
+                        text_current_offset += section.data.len();
+                    }
+                    ".data" => {
+                        data_offsets.insert((obj_idx, section_idx as u16), data_current_offset);
+                        raw_data_section.extend_from_slice(&section.data);
+                        data_current_offset += section.data.len();
+                    }
+                    _ => {}
+                }
+            }
+        }
+
+        // .textセクションの配置
+        // ELFヘッダー(64バイト) + プログラムヘッダー(56バイト×2) の後
+        let text_offset = 0x100;
+        let text_addr = align(base_addr + text_offset, 4);
+
+        let text_section = Section {
+            name: Cow::Borrowed(".text"),
+            r#type: SectionType::ProgBits,
+            flags: vec![SectionFlag::Alloc, SectionFlag::ExecInstr],
+            addr: text_addr,
+            offset: text_offset,
+            size: raw_text_section.len() as u64,
+            data: Cow::Owned(raw_text_section),
+            align: 4,
+        };
+
+        // .dataセクションの配置
+        let data_offset = text_offset + text_section.size;
+        // メモリ上は0x10000のギャップを設ける（ページ境界）
+        let data_base_addr = align(text_section.addr + text_section.size + 0x10000, 4);
+
+        let data_section = Section {
+            name: Cow::Borrowed(".data"),
+            r#type: SectionType::ProgBits,
+            flags: vec![SectionFlag::Alloc, SectionFlag::Write],
+            addr: data_base_addr,
+            offset: data_offset,
+            size: raw_data_section.len() as u64,
+            data: Cow::Owned(raw_data_section),
+            align: 4,
+        };
+
+        // シンボルのアドレスを更新
+        for symbol in resolved_symbols.values_mut() {
+            if let Some(&offset) = text_offsets.get(&(symbol.object_index, symbol.section_index)) {
+                symbol.value = text_section.addr + (offset as u64) + symbol.value;
+            } else if let Some(&offset) = data_offsets.get(&(symbol.object_index, symbol.section_index)) {
+                symbol.value = data_section.addr + (offset as u64) + symbol.value;
+            }
+        }
+
+        let mut output_sections = vec![text_section, data_section];
+
+        // 再配置を適用（次章で実装）
+        self.apply_relocations(&mut output_sections, resolved_symbols)?;
+
+        Ok(output_sections)
+    }
+
+    /// シンボルテーブルと文字列テーブルを作成
+    fn make_symbol_section(
+        &self,
+        offset: u64,
+        resolved_symbols: &HashMap<String, ResolvedSymbol>,
+    ) -> (Section<'static>, Section<'static>) {
+        use crate::elf::symbol::{Binding, SymbolType};
+
+        // 文字列テーブルを作成
+        let mut strtab: Vec<u8> = vec![0]; // NULL文字から始める
+        let mut symbol_name_offsets: HashMap<String, usize> = HashMap::new();
+
+        for name in resolved_symbols.keys() {
+            if name.is_empty() {
+                continue;
+            }
+            symbol_name_offsets.insert(name.clone(), strtab.len());
+            strtab.extend_from_slice(name.as_bytes());
+            strtab.push(0);
+        }
+
+        let strtab_section = Section {
+            name: Cow::Borrowed(".strtab"),
+            r#type: SectionType::StrTab,
+            flags: vec![],
+            addr: 0,
+            offset,
+            size: strtab.len() as u64,
+            data: Cow::Owned(strtab),
+            align: 1,
+        };
+
+        // シンボルテーブルを作成
+        let mut symtab: Vec<u8> = vec![];
+
+        // 最初のエントリはNULLシンボル（24バイト）
+        symtab.extend_from_slice(&[0u8; 24]);
+
+        // ローカルシンボル数をカウント
+        let mut local_count = 1u32; // NULLシンボル分
+
+        // ローカルシンボルを先に追加
+        for symbol in resolved_symbols.values() {
+            if symbol.info.binding != Binding::Local {
+                continue;
+            }
+            if symbol.name.is_empty() || symbol.info.r#type == SymbolType::File {
+                continue;
+            }
+            local_count += 1;
+            self.write_symbol_entry(&mut symtab, symbol, &symbol_name_offsets);
+        }
+
+        // グローバルシンボルを追加
+        for symbol in resolved_symbols.values() {
+            if symbol.info.binding == Binding::Local {
+                continue;
+            }
+            if symbol.name.is_empty() || symbol.info.r#type == SymbolType::File {
+                continue;
+            }
+            self.write_symbol_entry(&mut symtab, symbol, &symbol_name_offsets);
+        }
+
+        let symtab_offset = align(offset + strtab_section.size, 8);
+        let symtab_section = Section {
+            name: Cow::Borrowed(".symtab"),
+            r#type: SectionType::SymTab,
+            flags: vec![],
+            addr: 0,
+            offset: symtab_offset,
+            size: symtab.len() as u64,
+            data: Cow::Owned(symtab),
+            align: 8,
+        };

+        (symtab_section, strtab_section)
+    }
+
+    fn write_symbol_entry(
+        &self,
+        symtab: &mut Vec<u8>,
+        symbol: &ResolvedSymbol,
+        name_offsets: &HashMap<String, usize>,
+    ) {
+        use crate::elf::symbol::Binding;
+
+        // st_name (4バイト)
+        let name_offset = name_offsets.get(&symbol.name).copied().unwrap_or(0) as u32;
+        symtab.extend_from_slice(&name_offset.to_le_bytes());
+
+        // st_info (1バイト)
+        let binding = match symbol.info.binding {
+            Binding::Local => 0,
+            Binding::Global => 1,
+        };
+        let st_type = symbol.info.r#type as u8;
+        let st_info = (binding << 4) | st_type;
+        symtab.push(st_info);
+
+        // st_other (1バイト)
+        symtab.push(0);
+
+        // st_shndx (2バイト) - セクションインデックス（1=.text, 2=.data）
+        let shndx = if symbol.section_index == 0 {
+            0u16
+        } else {
+            // 簡略化: .textを1, .dataを2とする
+            if symbol.value >= 0x410000 {
+                2u16
+            } else {
+                1u16
+            }
+        };
+        symtab.extend_from_slice(&shndx.to_le_bytes());
+
+        // st_value (8バイト)
+        symtab.extend_from_slice(&symbol.value.to_le_bytes());
+
+        // st_size (8バイト)
+        symtab.extend_from_slice(&symbol.size.to_le_bytes());
+    }
+
+    // 次章で実装
+    pub fn apply_relocations(
+        &self,
+        _sections: &mut Vec<Section<'static>>,
+        _resolved_symbols: &HashMap<String, ResolvedSymbol>,
+    ) -> Result<()> {
+        Ok(())
     }
 }
```

### テストを実行する

```sh
$ cargo test linker::section::tests::layout_sections
running 1 test
test linker::section::tests::layout_sections_returns_sections_with_correct_addresses ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## シンボルアドレス更新の仕組み

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

## まとめ

本章ではセクション配置を実装した。

- `todo!()`でスタブを作成してからテストを書く
- `align`関数で値を2のべき乗の境界に揃える
- 複数オブジェクトの`.text`と`.data`をマージ
- マージ後のオフセットを記録し、シンボルアドレスを更新
- `.text`は`0x400100`、`.data`は`.text`の後に`0x10000`のギャップを設けて配置

次章では、再配置の適用を実装する。
