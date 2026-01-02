---
title: "再配置の適用"
---

本章では、再配置の適用を実装する。再配置は、シンボル参照を実際のメモリアドレスに書き換える処理である。

## 再配置の基本原理

再配置では次の処理を行う。

1. 再配置エントリを読み取る
2. 参照先シンボルのアドレスを取得
3. 命令のアドレスと参照先アドレスから計算
4. 命令の即値フィールドを書き換え

## R_AARCH64_ADR_PREL_LO21の処理

本書で扱う再配置タイプは`R_AARCH64_ADR_PREL_LO21`である。
これはARM64の`ADR`命令用のPC相対アドレスを計算する。

`main.o`の再配置情報を確認する。

```sh
$ readelf -r main.o
Relocation section '.rela.text' at offset 0x178 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000000  000900000112 R_AARCH64_ADR_PRE 0000000000000000 x + 0
```

これは「`.text`セクションのオフセット0にある命令で、シンボル`x`への参照がある」ことを示している。

## ARM64のADR命令

`ADR`命令は、PC（プログラムカウンタ）からの相対アドレスをレジスタにロードする命令である。

```
adr x0, x    ; 変数xのアドレスをx0にロード
```

### ADR命令のビットフィールド

ADR命令は32ビットで、次のようなビットフィールドを持つ。

```
 31 30 29 28 27 26 25 24 23                    5 4     0
┌──┬─────┬─────────────────────────────────────┬────────┐
│op│immlo│ 1  0  0  0  0 │       immhi         │   Rd   │
└──┴─────┴─────────────────────────────────────┴────────┘
```

- **op** (1ビット): オペコード（ADR=0, ADRP=1）
- **immlo** (2ビット): 即値の下位2ビット（ビット29-30）
- **immhi** (19ビット): 即値の上位19ビット（ビット5-23）
- **Rd** (5ビット): 出力レジスタ

21ビットの即値は`immhi`と`immlo`に分割されて格納される。

## 再配置の実装

```rust:src/linker/relocation.rs
use std::collections::HashMap;
use crate::elf::relocation::{RelocationAddend, RelocationType};
use crate::error::{LinkerError, Result};
use super::Linker;
use super::output::{ResolvedSymbol, Section};

impl Linker {
    pub fn apply_relocations(
        &self,
        output_sections: &mut [Section<'static>],
        resolved_symbols: &HashMap<String, ResolvedSymbol>,
    ) -> Result<()> {
        // セクション名からインデックスへのマップを作成
        let section_indices: HashMap<String, usize> = output_sections
            .iter()
            .enumerate()
            .map(|(i, sec)| (sec.name.to_string(), i))
            .collect();

        // 各オブジェクトファイルの再配置を処理
        for (obj_idx, obj) in self.objects.iter().enumerate() {
            for reloc in &obj.relocations {
                self.process_relocation(
                    obj_idx,
                    reloc,
                    output_sections,
                    &section_indices,
                    resolved_symbols,
                )?;
            }
        }

        Ok(())
    }

    fn process_relocation(
        &self,
        obj_idx: usize,
        reloc: &RelocationAddend,
        output_sections: &mut [Section<'static>],
        section_indices: &HashMap<String, usize>,
        resolved_symbols: &HashMap<String, ResolvedSymbol>,
    ) -> Result<()> {
        match reloc.info.r#type {
            RelocationType::Aarch64AdrPrelLo21 => {
                // シンボルインデックスを取得
                let symbol_index = reloc.info.symbol_index as usize;
                if symbol_index >= self.objects[obj_idx].symbols.len() {
                    return Err(LinkerError::RelocationError {
                        message: format!("Symbol index out of range: {}", symbol_index),
                    });
                }

                // シンボル名を取得
                let symbol_name = &self.objects[obj_idx].symbols[symbol_index].name;

                // 解決済みシンボルを取得
                let resolved_symbol = resolved_symbols
                    .get(symbol_name)
                    .ok_or_else(|| LinkerError::RelocationError {
                        message: format!("Symbol not resolved: {}", symbol_name),
                    })?;

                // .textセクションを取得
                let text_section_idx = section_indices
                    .get(".text")
                    .ok_or_else(|| LinkerError::SectionNotFound {
                        name: ".text".to_string(),
                    })?;

                let target_section = &mut output_sections[*text_section_idx];

                // 再配置オフセットの範囲チェック
                if reloc.offset as usize >= target_section.data.len() {
                    return Err(LinkerError::RelocationError {
                        message: format!("Offset out of range: {}", reloc.offset),
                    });
                }

                // アドレスを計算
                let instruction_addr = target_section.addr + reloc.offset;
                let symbol_addr = resolved_symbol.value;

                // 相対アドレスを計算
                // relative_addr = シンボルアドレス - 命令アドレス + addend
                let relative_addr =
                    ((symbol_addr as i64) - (instruction_addr as i64) + reloc.addend) as i32;

                // 命令を取得
                let pos = reloc.offset as usize;
                let data = target_section.data.to_mut();
                let instruction = u32::from_le_bytes(
                    data[pos..pos + 4].try_into().unwrap()
                );

                // オペコードとレジスタ部分を保持
                // ADR命令: ビット28-24が10000、ビット0-4がレジスタ
                let opcode_rd = instruction & 0x9F00001F;

                // 即値をエンコード
                // immlo: 下位2ビット → ビット29-30
                let immlo = ((relative_addr & 0x3) as u32) << 29;
                // immhi: 上位19ビット → ビット5-23
                let immhi = (((relative_addr >> 2) & 0x7FFFF) as u32) << 5;

                // 新しい命令を組み立て
                let new_instruction = opcode_rd | immlo | immhi;

                // 命令を書き換え
                data[pos..pos + 4].copy_from_slice(&new_instruction.to_le_bytes());
            }
            _ => {
                return Err(LinkerError::RelocationError {
                    message: format!("Unsupported relocation type: {:?}", reloc.info.r#type),
                });
            }
        }

        Ok(())
    }
}
```

## 再配置の計算例

具体的な計算例を見てみる。

**前提条件**:
- `_start`（命令のアドレス）: `0x400100`
- `x`（シンボルのアドレス）: `0x410110`
- `addend`: `0`

**相対アドレスの計算**:
```
relative_addr = symbol_addr - instruction_addr + addend
              = 0x410110 - 0x400100 + 0
              = 0x10010
```

**即値のエンコード**:
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

**新しい命令**:
```
opcode_rd     = 0x10000000  (ADR x0)
immlo         = 0x00000000
immhi         = 0x00080080
new_instruction = opcode_rd | immlo | immhi
                = 0x10080080
```

これでADR命令が正しくエンコードされ、実行時に`x0`レジスタに`x`のアドレスがロードされる。

次章では、ELF出力の実装を行う。
