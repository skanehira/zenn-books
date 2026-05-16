---
title: "再配置の適用"
---

本章では再配置を実装する。
シンボル参照を実際のメモリアドレスに書き換えていく処理で、リンカーの「最後の仕上げ」にあたる部分である。

## 本章で実装するファイル

```
src/
├── error.rs               # 本章（エラー追加）
├── linker.rs              # 本章（モジュール追加）
├── linker/
│   ├── relocation.rs      # 本章
│   └── section.rs         # 本章（スタブを削除）
└── ...
```

## モジュール構造を更新する

### linker.rsを更新する

```diff:src/linker.rs
 pub mod output;
+pub mod relocation;
 pub mod section;
 pub mod symbol;

 use crate::parser::Elf;
```

### 空のモジュールファイルを作成する

```sh
$ touch src/linker/relocation.rs
```

## 再配置の基本原理

再配置でやることはシンプルで、次の4ステップで構成される。

1. 再配置エントリを読み取る
2. 参照先シンボルのアドレスを取得する
3. 命令のアドレスと参照先アドレスから書き換える値を計算する
4. 命令の即値フィールドを書き換える

「どう書き換えるか」は再配置タイプによって変わるので、ここでは本書で扱う1種類だけを実装する。

## R_AARCH64_ADR_PREL_LO21の処理

本書で扱う再配置タイプは`R_AARCH64_ADR_PREL_LO21`、ARM64の`ADR`命令用のPC相対アドレスを計算するタイプである。

`main.o`の再配置情報を改めて確認しておく。

```sh
$ readelf -r main.o
Relocation section '.rela.text' at offset 0x178 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000000  000900000112 R_AARCH64_ADR_PRE 0000000000000000 x + 0
```

これは「`.text`セクションのオフセット0にある命令で、シンボル`x`への参照がある」という情報である。

## ARM64のADR命令

`ADR`命令は、PC（プログラムカウンタ）からの相対アドレスをレジスタにロードする命令である。

```
adr x0, x    ; 変数xのアドレスをx0にロード
```

### PCとは

PCは「現在CPUが実行している命令のメモリアドレス」を保持するレジスタである。
ADR命令を実行する瞬間、PCはそのADR命令自身が置かれているアドレスを指している。
したがって「PC相対アドレス」とは、その命令自身の位置を基準にした相対位置のことをいう。

```
メモリ
┌──────────────────────────────┐
│ ...                          │
├──────────────────────────────┤
│ 0x400100: adr x0, x  ← この命令を実行中なら PC = 0x400100
├──────────────────────────────┤
│ 0x400104: ldr w0, [x0]       │
├──────────────────────────────┤
│ ...                          │
├──────────────────────────────┤
│ 0x410110: x = 11             │
└──────────────────────────────┘
```

ADR命令にはこの「PCからの差分」が即値として埋め込まれていて、CPUは実行時に `PC + 即値` を計算してレジスタに入れる。

### ADR命令のビットフィールド

ADR命令は32ビット長で、ビットの内訳は次のようになっている。

```
 31 30 29 28 27 26 25 24 23                    5 4     0
┌──┬─────┬─────────────────────────────────────┬────────┐
│op│immlo│ 1  0  0  0  0 │       immhi         │   Rd   │
└──┴─────┴─────────────────────────────────────┴────────┘
```

- op (1ビット): オペコード（ADR=0, ADRP=1）
- immlo (2ビット): 即値の下位2ビット（ビット29-30）
- immhi (19ビット): 即値の上位19ビット（ビット5-23）
- Rd (5ビット): 出力レジスタ

21ビットの即値が`immlo`と`immhi`に分割して埋め込まれている。

## リンク時にPCを計算する

リンカーはこの命令に埋め込むべき「PC相対オフセット」を計算するために、実行時のPC値を事前に知る必要がある。

PCは本来CPU内部の値だが、リンカーは`.text`セクションをどの仮想アドレスに配置するかを自分で決めるので、各命令の実行時PCをリンク時に計算できる。

```
実行時PC = .text の仮想アドレス + セクション内オフセット
       = target_section.addr + reloc.offset
```

たとえば`.text`を`0x400100`に配置し、再配置対象の命令が`.text`の先頭（オフセット0）にあるなら、その命令の実行時PCは`0x400100`になる。

このPC値が手に入れば、あとは参照先シンボルのアドレスとの差分を取るだけで、命令に埋め込む相対アドレスが求まる。

```
相対アドレス = シンボルのアドレス - PC + addend
```

この計算結果を`immlo`と`immhi`に分解して命令に書き戻す、というのが再配置処理の中身になる。

## 再配置の計算例

実装に入る前に、ここまでで出てきた式を実際の値で一度回しておく。
これからRustで書く処理が何を計算しているのかが具体的にイメージできるようになる。

前提条件：

- `_start`（命令のアドレス）: `0x400100`
- `x`（シンボルのアドレス）: `0x410110`
- `addend`: `0`

相対アドレスの計算：

```
relative_addr = symbol_addr - instruction_addr + addend
              = 0x410110 - 0x400100 + 0
              = 0x10010
```

即値のエンコード：

```
relative_addr = 0x10010 = 0b0001_0000_0000_0001_0000

immlo = (0x10010 & 0x3) << 29
      = 0 << 29
      = 0

immhi = ((0x10010 >> 2) & 0x7FFFF) << 5
      = (0x4004 & 0x7FFFF) << 5
      = 0x4004 << 5
      = 0x80080
```

新しい命令：

```
opcode_rd     = 0x10000000  (ADR x0)
immlo         = 0x00000000
immhi         = 0x00080080
new_instruction = opcode_rd | immlo | immhi
                = 0x10080080
```

これでADR命令が正しくエンコードされ、実行時に`x0`レジスタに`x`のアドレスがロードされる。
次節からは、この計算をそのままRustに落としていく。

## エラー型の追加

```diff:src/error.rs
     #[error("IO error: {0}")]
     Io(#[from] std::io::Error),
+
+    #[error("Relocation error: {message}")]
+    RelocationError { message: String },
+
+    #[error("Section not found: {name}")]
+    SectionNotFound { name: String },
 }
```

## 再配置の実装

### テストを書く

ADR命令のオペコード部分（ビット28-24）に加え、リロケーションで埋め込まれる即値の`immlo`（ビット30-29）と`immhi`（ビット23-5）もチェックする。
前提として、`_start`の命令アドレスは`0x400100`、参照先`x`のアドレスは`0x410110`なので、相対アドレスは`0x10010`になる。これを下位2ビットと上位19ビットに分けた値が即値として埋め込まれているはずである。

```rust:src/linker/relocation.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::linker::Linker;
    use crate::parser::Elf;

    #[test]
    fn apply_relocations() {
        let main_o = include_bytes!("../parser/fixtures/main.o");
        let sub_o = include_bytes!("../parser/fixtures/sub.o");

        let mut linker = Linker::default();
        linker.add_object("main.o".to_string(), Elf::parse(main_o).unwrap());
        linker.add_object("sub.o".to_string(), Elf::parse(sub_o).unwrap());

        let mut resolved = linker.resolve_symbols().unwrap();
        let (sections, _) = linker.layout_sections(&mut resolved).unwrap();

        // .textセクションの最初の命令が書き換えられている
        let text = sections.iter().find(|s| s.name == ".text").unwrap();
        let instruction = u32::from_le_bytes(text.data[0..4].try_into().unwrap());

        // ADR命令のオペコード（ビット28-24）
        let opcode = (instruction >> 24) & 0x1F;
        assert_eq!(opcode, 0x10);

        // 即値の下位2ビット immlo（ビット30-29）
        // 0x400100 → 0x410110 の相対 0x10010 の下位2ビット = 0
        let immlo = (instruction >> 29) & 0x3;
        assert_eq!(immlo, 0);

        // 即値の上位19ビット immhi（ビット23-5）
        // 0x10010 を2ビット右シフトした 0x4004 が埋め込まれている
        let immhi = (instruction >> 5) & 0x7FFFF;
        assert_eq!(immhi, 0x4004);
    }
}
```

### 再配置処理を実装する

各オブジェクトの再配置エントリを順に処理し、種類ごとに分岐する形で書いていく。
今回扱うのは1種類だけなので分岐はほぼないが、構造としては「将来別のタイプを足しやすい形」にしてある。

実装の中で `let opcode_rd = instruction & 0x9F00001F;` というマスクが出てくるが、
これは ADR 命令の中で **「即値以外の部分」を残すためのマスク** である。

```diff:src/linker/relocation.rs
+use std::collections::HashMap;
+
+use crate::elf::relocation::RelocationType;
+use crate::error::{LinkerError, Result};
+
+use super::output::{ResolvedSymbol, Section};
+use super::Linker;
+
+impl Linker {
+    pub fn apply_relocations(
+        &self,
+        output_sections: &mut [Section<'static>],
+        resolved_symbols: &HashMap<String, ResolvedSymbol>,
+    ) -> Result<()> {
+        // セクション名からインデックスへのマップを作成
+        let section_indices: HashMap<String, usize> = output_sections
+            .iter()
+            .enumerate()
+            .map(|(i, sec)| (sec.name.to_string(), i))
+            .collect();
+
+        // 各オブジェクトファイルの再配置を処理
+        for (obj_idx, obj) in self.objects.iter().enumerate() {
+            for reloc in &obj.relocations {
+                self.process_relocation(
+                    obj_idx,
+                    reloc,
+                    output_sections,
+                    &section_indices,
+                    resolved_symbols,
+                )?;
+            }
+        }
+
+        Ok(())
+    }
+
+    fn process_relocation(
+        &self,
+        obj_idx: usize,
+        reloc: &crate::elf::relocation::Rela,
+        output_sections: &mut [Section<'static>],
+        section_indices: &HashMap<String, usize>,
+        resolved_symbols: &HashMap<String, ResolvedSymbol>,
+    ) -> Result<()> {
+        match reloc.info.r#type {
+            RelocationType::Aarch64AdrPrelLo21 => {
+                // シンボルインデックスを取得
+                let symbol_index = reloc.info.symbol_index as usize;
+                if symbol_index >= self.objects[obj_idx].symbols.len() {
+                    return Err(LinkerError::RelocationError {
+                        message: format!("Symbol index out of range: {}", symbol_index),
+                    });
+                }
+
+                // シンボル名を取得
+                let symbol_name = &self.objects[obj_idx].symbols[symbol_index].name;
+
+                // 解決済みシンボルを取得
+                let resolved_symbol = resolved_symbols
+                    .get(symbol_name)
+                    .ok_or_else(|| LinkerError::RelocationError {
+                        message: format!("Symbol not resolved: {}", symbol_name),
+                    })?;
+
+                // .textセクションを取得
+                let text_section_idx = section_indices
+                    .get(".text")
+                    .ok_or_else(|| LinkerError::SectionNotFound {
+                        name: ".text".to_string(),
+                    })?;
+
+                let target_section = &mut output_sections[*text_section_idx];
+
+                // 再配置オフセットの範囲チェック
+                if reloc.offset as usize >= target_section.data.len() {
+                    return Err(LinkerError::RelocationError {
+                        message: format!("Offset out of range: {}", reloc.offset),
+                    });
+                }
+
+                // アドレスを計算
+                let instruction_addr = target_section.addr + reloc.offset;
+                let symbol_addr = resolved_symbol.value;
+
+                // 相対アドレスを計算
+                // relative_addr = シンボルアドレス - 命令アドレス + addend
+                let relative_addr =
+                    ((symbol_addr as i64) - (instruction_addr as i64) + reloc.addend) as i32;
+
+                // 命令を取得
+                let pos = reloc.offset as usize;
+                let data = target_section.data.to_mut();
+                let instruction =
+                    u32::from_le_bytes(data[pos..pos + 4].try_into().unwrap());
+
+                // オペコードとレジスタ部分を保持
+                // ADR命令: ビット28-24が10000、ビット0-4がレジスタ
+                let opcode_rd = instruction & 0x9F00001F;
+
+                // 即値をエンコード
+                // immlo: 下位2ビット → ビット29-30
+                let immlo = ((relative_addr & 0x3) as u32) << 29;
+                // immhi: 上位19ビット → ビット5-23
+                let immhi = (((relative_addr >> 2) & 0x7FFFF) as u32) << 5;
+
+                // 新しい命令を組み立て
+                let new_instruction = opcode_rd | immlo | immhi;
+
+                // 命令を書き換え
+                data[pos..pos + 4].copy_from_slice(&new_instruction.to_le_bytes());
+            }
+            _ => {
+                return Err(LinkerError::RelocationError {
+                    message: format!("Unsupported relocation type: {:?}", reloc.info.r#type),
+                });
+            }
+        }
+
+        Ok(())
+    }
+}
+
 #[cfg(test)]
 mod tests {
```

### section.rsのスタブを削除する

前章で仮置きしていた`apply_relocations`のスタブはもう不要なので消しておく。

```diff:src/linker/section.rs
-    // 次章で実装
-    pub fn apply_relocations(
-        &self,
-        _sections: &mut Vec<Section<'static>>,
-        _resolved_symbols: &HashMap<String, ResolvedSymbol>,
-    ) -> Result<()> {
-        Ok(())
-    }
 }
```

### テストを実行する

```sh
$ cargo test linker::relocation
running 1 test
test linker::relocation::tests::apply_relocations ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## まとめ

本章では再配置を実装した。

- `R_AARCH64_ADR_PREL_LO21`は21ビットのPC相対アドレスをエンコードする
- 即値は`immlo`（下位2ビット）と`immhi`（上位19ビット）に分割して埋め込む
- 相対アドレス = シンボルアドレス − 命令アドレス + addend

ここまででリンク処理本体は完成した。あとは結果をELFファイルとして書き出すだけである。次章では、その出力部分を実装していく。
