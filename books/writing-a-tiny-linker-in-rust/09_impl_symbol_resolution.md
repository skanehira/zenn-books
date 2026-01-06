---
title: "シンボル解決の実装"
---

本章では、シンボル解決の実装を行う。シンボル解決はリンカーの最も重要な処理の1つである。

## 本章で実装するファイル

```
src/
├── error.rs           # 本章
├── linker/
│   ├── mod.rs         # 本章
│   ├── output.rs      # 本章
│   └── symbol.rs      # 本章
└── lib.rs             # 本章（モジュール追加）
```

## モジュール構造を作成する

LSPが正しく動作するように、最初にモジュール宣言と空のファイルを作成する。

### lib.rsを更新する

```diff:src/lib.rs
 pub mod elf;
+pub mod error;
+pub mod linker;
 pub mod parser;
```

### linker/mod.rsを作成する

```rust:src/linker/mod.rs
pub mod output;
pub mod symbol;
```

### 空のモジュールファイルを作成する

```sh
$ touch src/error.rs
$ mkdir -p src/linker
$ touch src/linker/mod.rs src/linker/output.rs src/linker/symbol.rs
```

## エラー型の定義

### テストを書く

`src/error.rs`にテストを書く。

```rust:src/error.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn error_display() {
        let err = LinkerError::UnresolvedSymbols {
            symbols: vec!["foo".to_string()],
        };
        assert_eq!(format!("{}", err), "Unresolved symbols: [\"foo\"]");
    }
}
```

### エラー型を実装する

```diff:src/error.rs
+use thiserror::Error;
+
+#[derive(Debug, Error)]
+pub enum LinkerError {
+    #[error("Duplicate symbols: {symbols:?}")]
+    DuplicateSymbol { symbols: Vec<String> },
+
+    #[error("Unresolved symbols: {symbols:?}")]
+    UnresolvedSymbols { symbols: Vec<String> },
+
+    #[error("Missing entry point: _start")]
+    MissingEntryPoint,
+
+    #[error("Parse error: {0}")]
+    Parse(String),
+
+    #[error("IO error: {0}")]
+    Io(#[from] std::io::Error),
+}
+
+pub type Result<T> = std::result::Result<T, LinkerError>;
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test error::tests::error_display
running 1 test
test error::tests::error_display ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Linker構造体の定義

### テストを書く

`src/linker/mod.rs`にテストを追加する。

```diff:src/linker/mod.rs
 pub mod output;
 pub mod symbol;
+
+#[cfg(test)]
+mod tests {
+    use super::*;
+
+    #[test]
+    fn add_object() {
+        let raw = include_bytes!("../parser/fixtures/sub.o");
+        let elf = crate::parser::Elf::parse(raw).unwrap();
+
+        let mut linker = Linker::new();
+        linker.add_object("sub.o".to_string(), elf);
+
+        assert_eq!(linker.objects.len(), 1);
+        assert_eq!(linker.object_names[0], "sub.o");
+    }
+}
```

### Linker構造体を実装する

```diff:src/linker/mod.rs
 pub mod output;
 pub mod symbol;

+use crate::parser::Elf;
+
+#[derive(Debug, Default)]
+pub struct Linker {
+    pub objects: Vec<Elf>,
+    pub object_names: Vec<String>,
+}
+
+impl Linker {
+    pub fn new() -> Self {
+        Self::default()
+    }
+
+    pub fn add_object(&mut self, name: String, obj: Elf) {
+        self.object_names.push(name);
+        self.objects.push(obj);
+    }
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test linker::tests::add_object
running 1 test
test linker::tests::add_object ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## 解決済みシンボルの構造体

### テストを書く

`src/linker/output.rs`にテストを書く。

```rust:src/linker/output.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::elf::symbol::{Binding, SymbolType};

    fn make_symbol(binding: Binding, is_defined: bool) -> ResolvedSymbol {
        ResolvedSymbol {
            name: "test".to_string(),
            value: 0,
            size: 0,
            info: symbol::Info {
                binding,
                r#type: SymbolType::NoType,
            },
            shndx: if is_defined { 1 } else { 0 },
            object_index: 0,
            is_defined,
        }
    }

    #[test]
    fn global_is_stronger_than_weak() {
        let global = make_symbol(Binding::Global, true);
        let weak = make_symbol(Binding::Weak, true);
        assert!(global.is_stronger_than(&weak));
    }

    #[test]
    fn local_is_stronger_than_global() {
        let local = make_symbol(Binding::Local, true);
        let global = make_symbol(Binding::Global, true);
        assert!(local.is_stronger_than(&global));
    }

    #[test]
    fn global_is_not_stronger_than_global() {
        let global1 = make_symbol(Binding::Global, true);
        let global2 = make_symbol(Binding::Global, true);
        assert!(!global1.is_stronger_than(&global2));
    }
}
```

バインディングの強さは `Local > Global > Weak` の順である。

- **Local**: ファイル内のみで有効。他のファイルから参照されない
- **Global**: 他のファイルから参照可能
- **Weak**: 同名のGlobalシンボルがあれば置き換えられる

### ResolvedSymbol構造体を実装する

```diff:src/linker/output.rs
+use crate::elf::symbol;
+
+#[derive(Debug, Clone)]
+pub struct ResolvedSymbol {
+    pub name: String,
+    pub value: u64,
+    pub size: u64,
+    pub info: symbol::Info,
+    pub shndx: u16,
+    pub object_index: usize,
+    pub is_defined: bool,
+}
+
+impl ResolvedSymbol {
+    /// シンボルの「強さ」を比較する
+    /// Local > Global > Weak の順で強い
+    pub fn is_stronger_than(&self, other: &Self) -> bool {
+        match (self.info.binding, other.info.binding) {
+            // LOCALは最も強い
+            (symbol::Binding::Local, _) => true,
+            // WEAKより他のすべてのバインディングは強い
+            (_, symbol::Binding::Weak) => true,
+            // 同じGLOBAL同士なら後勝ちしない
+            (symbol::Binding::Global, symbol::Binding::Global) => false,
+            // LOCALのほうがGLOBALより強い
+            (symbol::Binding::Global, symbol::Binding::Local) => false,
+            // WEAKは最も弱い
+            (symbol::Binding::Weak, _) => false,
+        }
+    }
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test linker::output
running 3 tests
test linker::output::tests::global_is_not_stronger_than_global ... ok
test linker::output::tests::global_is_stronger_than_weak ... ok
test linker::output::tests::local_is_stronger_than_global ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## シンボル解決の実装

### テストを書く

`src/linker/symbol.rs`にテストを書く。

```rust:src/linker/symbol.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::parser::Elf;

    #[test]
    fn resolve_symbols() {
        let main_o = include_bytes!("../parser/fixtures/main.o");
        let sub_o = include_bytes!("../parser/fixtures/sub.o");

        let mut linker = Linker::new();
        linker.add_object("main.o".to_string(), Elf::parse(main_o).unwrap());
        linker.add_object("sub.o".to_string(), Elf::parse(sub_o).unwrap());

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

    #[test]
    fn unresolved_symbol_error() {
        let main_o = include_bytes!("../parser/fixtures/main.o");

        let mut linker = Linker::new();
        linker.add_object("main.o".to_string(), Elf::parse(main_o).unwrap());

        // sub.oがないのでxが未解決
        let result = linker.resolve_symbols();
        assert!(matches!(
            result,
            Err(LinkerError::UnresolvedSymbols { .. })
        ));
    }
}
```

### シンボル解決の流れ

`main.o`と`sub.o`のシンボルテーブルを例に、シンボル解決の流れを説明する。

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

### パース処理を実装する

```diff:src/linker/symbol.rs
+use std::collections::HashMap;
+
+use crate::elf::symbol::SHN_UNDEF;
+use crate::error::{LinkerError, Result};
+
+use super::output::ResolvedSymbol;
+use super::Linker;
+
+impl Linker {
+    pub fn resolve_symbols(&self) -> Result<HashMap<String, ResolvedSymbol>> {
+        let mut resolved_symbols: HashMap<String, ResolvedSymbol> = HashMap::new();
+        let mut duplicate_symbols = vec![];
+
+        // すべてのオブジェクトファイル内のシンボルを処理
+        for (obj_idx, obj) in self.objects.iter().enumerate() {
+            for symbol in &obj.symbols {
+                let new_symbol = ResolvedSymbol {
+                    name: symbol.name.clone(),
+                    value: symbol.value,
+                    size: symbol.size,
+                    info: symbol.info,
+                    shndx: symbol.shndx,
+                    object_index: obj_idx,
+                    // shndxがSHN_UNDEF(0)でなければ定義済み
+                    is_defined: symbol.shndx != SHN_UNDEF,
+                };
+
+                // 同名シンボルが既にあるか確認
+                if let Some(existing) = resolved_symbols.get(&symbol.name) {
+                    // 両方が定義済みの場合
+                    if new_symbol.is_defined && existing.is_defined {
+                        // 新しいシンボルの方が強ければ上書き
+                        if new_symbol.is_stronger_than(existing) {
+                            resolved_symbols.insert(symbol.name.clone(), new_symbol);
+                        } else {
+                            // 同じ強さなら重複エラー
+                            duplicate_symbols.push(symbol.name.clone());
+                        }
+                    } else if new_symbol.is_defined && !existing.is_defined {
+                        // 新しいシンボルが定義済みで既存が未定義なら上書き
+                        // 例: main.oの未定義シンボルxがsub.oの定義済みxで解決
+                        resolved_symbols.insert(symbol.name.clone(), new_symbol);
+                    }
+                    // 既存が定義済みで新しいのが未定義なら何もしない
+                } else {
+                    // 新規シンボルを追加
+                    resolved_symbols.insert(symbol.name.clone(), new_symbol);
+                }
+            }
+        }
+
+        // 重複シンボルがあればエラー
+        if !duplicate_symbols.is_empty() {
+            return Err(LinkerError::DuplicateSymbol {
+                symbols: duplicate_symbols,
+            });
+        }
+
+        // 未解決シンボルを抽出
+        let unresolved_symbols: Vec<_> = resolved_symbols
+            .iter()
+            .filter_map(|(_, symbol)| {
+                if symbol.is_defined {
+                    return None;
+                }
+                Some(symbol.name.clone())
+            })
+            .collect();
+
+        // 未解決シンボルがあればエラー
+        if !unresolved_symbols.is_empty() {
+            return Err(LinkerError::UnresolvedSymbols {
+                symbols: unresolved_symbols,
+            });
+        }
+
+        Ok(resolved_symbols)
+    }
+}
+
 #[cfg(test)]
 mod tests {
```

### テストを実行する

```sh
$ cargo test linker::symbol
running 2 tests
test linker::symbol::tests::resolve_symbols ... ok
test linker::symbol::tests::unresolved_symbol_error ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## まとめ

本章ではシンボル解決を実装した。

- モジュール宣言を先に追加してLSPを有効化
- シンボルの強さは `Local > Global > Weak` の順
- 未定義シンボルは定義済みシンボルで上書きされる
- 同じ強さの定義済みシンボルが重複するとエラー
- 解決されないシンボルが残るとエラー

次章では、セクション配置の実装を行う。
