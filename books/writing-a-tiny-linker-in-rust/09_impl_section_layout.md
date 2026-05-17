---
title: "セクション配置の実装"
---

本章ではセクション配置を実装していく。
各オブジェクトファイルから`.text`や`.data`を集めてマージし、それぞれをメモリ上のどのアドレスに置くかを決める処理である。

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

リンク後のセクションを表す`Section`構造体を追加する。
パース時の`elf::section::Header`とは別物で、こちらは「最終的にファイルに書き出すセクション」のための型である。

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

## アラインメント関数

セクションのアドレスを計算するときに、よく「指定した境界に切り上げる」という処理が出てくる。
ここでまとめて`align`関数として用意しておく。

### テストを書く

`src/linker/section.rs`にテストを書く。

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

`alignment`が2のべき乗であることを前提にしたビット演算で書ける。

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

ここからが本章の本題である。本書のスタイルに従って、まずテストを書き、次に実装の意図を図と式で整理し、最後にコードを積み上げていく流れで進める。

### テストを書く

ここでは「最終的にどんなアドレスに配置されているか」を検証するテストを先に書く。
期待値の `0x400100` や `0x10000` のギャップといった具体的な数値の根拠は、テスト記述のあとの「セクション配置を実装する」節で図と式を交えて説明するので、ここでは「あとで腑に落ちる」前提で読み進めてほしい。

なお、ここで導入する定数 `BASE_ADDR` は、07章で説明した実行ファイルのベースアドレス `0x400000` をモジュール定数として置いたものである。

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
+/// 実行ファイルのベースアドレス（07章で説明したAArch64 Linuxの慣習値）
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
+        let mut linker = Linker::default();
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

ここからが本題のセクション配置である。実装は長めになるので、まず「マージ後オフセット」というデータ構造と、シンボルアドレスを決める計算式を先に押さえてから、コードを4ステップに分けて積み上げていく。

#### マージ後オフセットの記録

複数のオブジェクトファイルから`.text`を集めて1本にマージするとき、
**「元の (オブジェクト, セクション) がマージ後の何バイト目に置かれたか」** を覚えておく必要がある。
そうしないと、シンボルが「もとのセクション内オフセット」しか持っていないので、マージ後の絶対アドレスに変換できない。

たとえば図1のような状況を考える。

![](/images/writing-a-tiny-linker-in-rust/ch09-01-section-merge.png)
*図1*

このとき:

- `main.o`の`.text`はマージ後`0`バイト目から
- `sub.o`の`.text`はマージ後`16`バイト目から

これを覚えておくために、本書では次のような`HashMap`を使う。

```rust
// キー: (オブジェクトのインデックス, 元のセクションインデックス)
// 値:   マージ後の .text 内オフセット
let mut text_offsets: HashMap<(usize, u16), usize> = HashMap::new();
```

`.data`についても同じ仕組みで`data_offsets`を作る。

#### シンボルアドレス更新の仕組み

マージ後オフセットが揃ったら、最後にシンボルが指すアドレスを元のオブジェクトファイル内のオフセットから最終的なメモリアドレスに置き換える。
最終アドレスは図2のように、3つの値の足し算で求められる。

![](/images/writing-a-tiny-linker-in-rust/ch09-02-symbol-address-calculation.png)
*図2*

具体例で見ると次のとおり。`sub.o`の変数`x`は、もともと`.data`セクションのオフセット0にあるので、

```
x のアドレス = .dataセクションのアドレス + sub.oの.dataオフセット + 元のオフセット
            = 0x410110 + 0 + 0
            = 0x410110
```

`main.o`の`_start`は`.text`セクションのオフセット0にあるので、

```
_start のアドレス = .textセクションのアドレス + main.oの.textオフセット + 元のオフセット
                 = 0x400100 + 0 + 0
                 = 0x400100
```

となる。これから実装する`merge_sections`は、まさにこの式を実装するのが目的である。

#### 実装ステップ

`layout_sections`本体に加えて、補助関数を3つ用意する都合で実装はやや長くなる。
そのため本書では、次の4ステップに分けてコードを積み上げていく。テストが通るのはすべてのステップが揃った時点である点に注意してほしい。

1. `merge_sections`：`.text`と`.data`をマージし、シンボルアドレスを更新する
2. `make_symbol_section`と`write_symbol_entry`：シンボルテーブルと文字列テーブルを作る
3. `layout_sections`：上記2つの helper を呼んで全体を束ねる
4. `apply_relocations`のスタブ：次章で実装するための空関数

#### Step 1: merge_sections

`merge_sections` は各オブジェクトファイルの `.text` と `.data` をそれぞれ1本にマージし、その流れでシンボルのアドレスも更新する。後半の `for symbol in resolved_symbols.values_mut()` のループが、前節（図2）で示した「ベースアドレス + マージ後オフセット + 元のオフセット」の足し算をそのまま実装している部分である。
末尾で `self.apply_relocations(...)` を呼んでいるが、その本体は次章で書く。本章では Step 4 で空のスタブだけ用意するので、ここでは「呼び出す側がある」程度に思って読み進めてほしい。

```diff:src/linker/section.rs
 impl Linker {
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
+        // シンボルのアドレスを更新（図2の式そのもの）
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
+        // 再配置を適用（中身は次章で実装、今は Step 4 のスタブが呼ばれる）
+        self.apply_relocations(&mut output_sections, resolved_symbols)?;
+
+        Ok(output_sections)
+    }
 }
```

#### Step 2: make_symbol_section / write_symbol_entry

シンボルテーブル `.symtab` と、その中で参照される文字列テーブル `.strtab` を作る。ELFの規約で `.symtab` の中はローカルシンボルが先に並んでいる必要があるので、ローカル → グローバルの順で2回ループしている。
`write_symbol_entry` は1シンボルぶん（24バイト）を書き出すヘルパーで、03章で見た `st_info` の「上位4ビット = バインディング / 下位4ビット = タイプ」のビット詰め込みを、ここで逆方向に組み立てている。

```diff:src/linker/section.rs
 impl Linker {
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
+        // ローカルシンボルを先に追加
+        for symbol in resolved_symbols.values() {
+            if symbol.info.binding != Binding::Local {
+                continue;
+            }
+            if symbol.name.is_empty() || symbol.info.r#type == SymbolType::File {
+                continue;
+            }
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
+
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
 }
```

:::message
セクションインデックスを`symbol.value >= 0x410000`で判定しているのは簡略化のためで、複数の`.text`セクションや`.rodata`などを追加した瞬間に破綻する。
本格的には`ResolvedSymbol`側に出力セクションのインデックスを持たせ、`merge_sections`の中で確定値を書き戻す方式に拡張するのが本筋である。
:::

#### Step 3: layout_sections で全体を束ねる

ここまでで作った2つの helper を呼んで、最後にセクション名の文字列テーブル `.shstrtab` を組み立てる。
`.shstrtab` は「セクション名そのもの」を格納するNULL終端文字列の連続で、各セクションのヘッダーの `sh_name` フィールドが、この中のオフセットを指す。

セクションを並べるときの順序にも意図がある。最後に `.strtab` → `.symtab` → `.shstrtab` の順で push しているのは、`.symtab` のセクションヘッダーが持つ `sh_link` フィールドで `.strtab` のインデックスを指す必要があるためで、`.strtab` のほうが若いインデックスになるよう先に置いている（具体的な書き出しは11章）。

```diff:src/linker/section.rs
 impl Linker {
     pub fn layout_sections(
         &self,
-        _resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
+        resolved_symbols: &mut HashMap<String, ResolvedSymbol>,
     ) -> Result<(Vec<Section<'static>>, HashMap<String, usize>)> {
-        todo!()
+        // セクションのマージ（Step 1）
+        let output_sections = self.merge_sections(resolved_symbols, BASE_ADDR)?;
+
+        // シンボルテーブルと文字列テーブルの作成（Step 2）
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
 }
```

#### Step 4: apply_relocations のスタブ

再配置の本実装は次章で行うため、ここではコンパイルを通すための空関数だけ用意する。これで Step 1 の `merge_sections` から呼ばれても、何もせずにそのまま `Ok(())` を返すだけになる。

```diff:src/linker/section.rs
 impl Linker {
+    // 次章で実装
+    pub fn apply_relocations(
+        &self,
+        _sections: &mut [Section<'static>],
+        _resolved_symbols: &HashMap<String, ResolvedSymbol>,
+    ) -> Result<()> {
+        Ok(())
+    }
 }
```

### テストを実行する

```sh
$ cargo test linker::section::tests::layout_sections
running 1 test
test linker::section::tests::layout_sections_returns_sections_with_correct_addresses ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## まとめ

本章ではセクション配置を実装した。

- `align`関数で値を2のべき乗の境界に切り上げる
- 複数オブジェクトの`.text`/`.data`をマージする
- マージ後のオフセットを記録しておき、シンボルアドレスを更新する
- `.text`は`0x400100`、`.data`は`.text`のあとに`0x10000`のギャップを空けて配置する

ここまでで「どこに何を置くか」が決まった。次章では、その情報を使って実際にシンボル参照を実アドレスに書き換える、再配置を実装していく。
