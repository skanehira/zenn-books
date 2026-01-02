---
title: "シンボル解決の実装"
---

本章では、シンボル解決の実装を行う。シンボル解決はリンカーの最も重要な処理の1つである。

## Linker構造体の設計

まず、リンカーの状態を管理する構造体を定義する。

```rust:src/linker/mod.rs
pub mod output;
pub mod relocation;
pub mod section;
pub mod symbol;
pub mod writer;

use crate::parser::ELF;

#[derive(Debug, Default)]
pub struct Linker {
    objects: Vec<ELF>,
    object_names: Vec<String>,
}

impl Linker {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn add_object(&mut self, name: String, obj: ELF) {
        self.object_names.push(name);
        self.objects.push(obj);
    }
}
```

## 解決済みシンボルの構造体

シンボル解決の結果を格納する構造体を定義する。

```rust:src/linker/output.rs
use crate::elf::symbol;

#[derive(Debug, Clone)]
pub struct ResolvedSymbol {
    pub name: String,
    pub value: u64,
    pub size: u64,
    pub info: symbol::Info,
    pub shndx: u16,
    pub object_index: usize,
    pub is_defined: bool,
}

impl ResolvedSymbol {
    /// シンボルの「強さ」を比較する
    /// Local > Global > Weak の順で強い
    pub fn is_stronger_than(&self, other: &Self) -> bool {
        match (self.info.binding, other.info.binding) {
            // LOCALは最も強い
            (symbol::Binding::Local, _) => true,
            // WEAKより他のすべてのバインディングは強い
            (_, symbol::Binding::Weak) => true,
            // 同じGLOBAL同士なら後勝ちしない
            (symbol::Binding::Global, symbol::Binding::Global) => false,
            // LOCALのほうがGLOBALより強い
            (symbol::Binding::Global, symbol::Binding::Local) => false,
            // WEAKは最も弱い
            (symbol::Binding::Weak, _) => false,
            _ => false,
        }
    }
}
```

バインディングの強さは `Local > Global > Weak` の順である。

- **Local**: ファイル内のみで有効。他のファイルから参照されない
- **Global**: 他のファイルから参照可能
- **Weak**: 同名のGlobalシンボルがあれば置き換えられる

## シンボル解決の実装

シンボル解決のアルゴリズムを実装する。

```rust:src/linker/symbol.rs
use std::collections::HashMap;
use crate::elf::symbol::SHN_UNDEF;
use crate::error::{LinkerError, Result};
use super::Linker;
use super::output::ResolvedSymbol;

impl Linker {
    pub fn resolve_symbols(&self) -> Result<HashMap<String, ResolvedSymbol>> {
        let mut resolved_symbols: HashMap<String, ResolvedSymbol> = HashMap::new();
        let mut duplicate_symbols = vec![];

        // すべてのオブジェクトファイル内のシンボルを処理
        for (obj_idx, obj) in self.objects.iter().enumerate() {
            for symbol in &obj.symbols {
                let new_symbol = ResolvedSymbol {
                    name: symbol.name.clone(),
                    value: symbol.value,
                    size: symbol.size,
                    info: symbol.info,
                    shndx: symbol.shndx,
                    object_index: obj_idx,
                    // shndxがSHN_UNDEF(0)でなければ定義済み
                    is_defined: symbol.shndx != SHN_UNDEF,
                };

                // 同名シンボルが既にあるか確認
                if let Some(existing) = resolved_symbols.get(&symbol.name) {
                    // 両方が定義済みの場合
                    if new_symbol.is_defined && existing.is_defined {
                        // 新しいシンボルの方が強ければ上書き
                        if new_symbol.is_stronger_than(existing) {
                            resolved_symbols.insert(symbol.name.clone(), new_symbol);
                        } else {
                            // 同じ強さなら重複エラー
                            duplicate_symbols.push(symbol.name.clone());
                        }
                    } else if new_symbol.is_defined && !existing.is_defined {
                        // 新しいシンボルが定義済みで既存が未定義なら上書き
                        // 例: main.oの未定義シンボルxがsub.oの定義済みxで解決
                        resolved_symbols.insert(symbol.name.clone(), new_symbol);
                    }
                    // 既存が定義済みで新しいのが未定義なら何もしない
                } else {
                    // 新規シンボルを追加
                    resolved_symbols.insert(symbol.name.clone(), new_symbol);
                }
            }
        }

        // 重複シンボルがあればエラー
        if !duplicate_symbols.is_empty() {
            return Err(LinkerError::DuplicateSymbol {
                symbols: duplicate_symbols,
            });
        }

        // 未解決シンボルを抽出
        let unresolved_symbols: Vec<_> = resolved_symbols
            .iter()
            .filter_map(|(_, symbol)| {
                if symbol.is_defined {
                    return None;
                }
                Some(symbol.name.clone())
            })
            .collect();

        // 未解決シンボルがあればエラー
        if !unresolved_symbols.is_empty() {
            return Err(LinkerError::UnresolvedSymbols {
                symbols: unresolved_symbols,
            });
        }

        Ok(resolved_symbols)
    }
}
```

## シンボル解決の流れ

`main.o`と`sub.o`のシンボルテーブルを例に、シンボル解決の流れを見ていく。

**main.oのシンボル**:
- `_start`: GLOBAL, 定義済み（.textセクション）
- `x`: GLOBAL, 未定義（UND）

**sub.oのシンボル**:
- `x`: GLOBAL, 定義済み（.dataセクション）

シンボル解決の処理:

1. `main.o`の`_start`を追加（定義済み）
2. `main.o`の`x`を追加（未定義）
3. `sub.o`の`x`を処理
   - 既存の`x`は未定義、新しい`x`は定義済み
   - 定義済みの方で上書き → **解決成功**

最終的に:
- `_start`: `main.o`で定義
- `x`: `sub.o`で定義

## エラーハンドリング

リンカーで発生するエラーを定義する。

```rust:src/error.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum LinkerError {
    #[error("Duplicate symbols: {symbols:?}")]
    DuplicateSymbol { symbols: Vec<String> },

    #[error("Unresolved symbols: {symbols:?}")]
    UnresolvedSymbols { symbols: Vec<String> },

    #[error("Missing entry point: _start")]
    MissingEntryPoint,

    #[error("Parse error: {0}")]
    Parse(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

pub type Result<T> = std::result::Result<T, LinkerError>;
```

## テスト

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::parser::parse_elf;

    #[test]
    fn should_resolve_symbols() {
        let main_o = include_bytes!("../parser/fixtures/main.o");
        let sub_o = include_bytes!("../parser/fixtures/sub.o");

        let mut linker = Linker::new();
        linker.add_object("main.o".into(), parse_elf(main_o).unwrap().1);
        linker.add_object("sub.o".into(), parse_elf(sub_o).unwrap().1);

        let resolved = linker.resolve_symbols().unwrap();

        // _startが解決されている
        let start = resolved.get("_start").unwrap();
        assert!(start.is_defined);
        assert_eq!(start.object_index, 0); // main.o

        // xが解決されている
        let x = resolved.get("x").unwrap();
        assert!(x.is_defined);
        assert_eq!(x.object_index, 1); // sub.o
    }
}
```

これでシンボル解決の実装が完了した。次章では、セクション配置の実装を行う。
