---
title: "シンボル解決の実装"
---

本章ではシンボル解決を実装していく。
リンカーの肝となる処理の1つで、ここがちゃんと動かないとそもそもリンクが成立しない、というくらい重要な部分である。

## 本章で実装するファイル

```
src/
├── error.rs           # 本章
├── linker.rs          # 本章
├── linker/
│   ├── output.rs      # 本章
│   └── symbol.rs      # 本章
└── lib.rs             # 本章（モジュール追加）
```

## モジュール構造を作成する

### lib.rsを更新する

```diff:src/lib.rs
 pub mod elf;
+pub mod error;
+pub mod linker;
 pub mod parser;
```

### 空のモジュールファイルを作成する

```sh
$ touch src/error.rs src/linker.rs
$ mkdir -p src/linker
$ touch src/linker/output.rs src/linker/symbol.rs
```

### linker.rsを作成する

```rust:src/linker.rs
pub mod output;
pub mod symbol;
```

## エラー型の定義

リンカー全体で使うエラー型を`src/error.rs`に定義しておく。
シンボル解決だけでなく、後続の章で出てくるIOやパースのエラーもまとめて入れておく。

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

## Linker構造体の定義

リンカー本体の入れ物になる`Linker`構造体を作る。
今のところは入力のオブジェクトファイル一覧と、それぞれの名前（エラーメッセージ用）を持つだけのシンプルな構造である。

```rust:src/linker.rs
pub mod output;
pub mod symbol;

use crate::parser::Elf;

#[derive(Debug, Default)]
pub struct Linker {
    pub objects: Vec<Elf>,
    pub object_names: Vec<String>,
}

impl Linker {
    pub fn add_object(&mut self, name: String, obj: Elf) {
        self.object_names.push(name);
        self.objects.push(obj);
    }
}
```

## 解決済みシンボルの構造体

シンボル解決の結果を入れるための`ResolvedSymbol`構造体を用意する。
元の`Symbol`との違いは、どのオブジェクトに属するかや、定義済みかどうかといった「リンカーが解決過程で必要になる情報」を持っている点である。

### テストを書く

`src/linker/output.rs`に`ResolvedSymbol`構造体のスタブとテストを追加する。

```rust:src/linker/output.rs
use crate::elf::symbol::{self, Binding};

#[derive(Debug, Clone)]
pub struct ResolvedSymbol {
    pub name: String,
    pub value: u64,
    pub size: u64,
    pub info: symbol::Info,
    pub section_index: u16,
    pub object_index: usize,
    pub is_defined: bool,
}

impl ResolvedSymbol {
    pub fn is_stronger_than(&self, other: &Self) -> bool {
        todo!()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::elf::symbol::SymbolType;

    fn make_symbol(binding: Binding, is_defined: bool) -> ResolvedSymbol {
        ResolvedSymbol {
            name: "test".to_string(),
            value: 0,
            size: 0,
            info: symbol::Info {
                binding,
                r#type: SymbolType::NoType,
            },
            section_index: if is_defined { 1 } else { 0 },
            object_index: 0,
            is_defined,
        }
    }

    #[test]
    fn is_stronger_than_returns_true_when_local_vs_global() {
        let local = make_symbol(Binding::Local, true);
        let global = make_symbol(Binding::Global, true);
        assert!(local.is_stronger_than(&global));
    }

    #[test]
    fn is_stronger_than_returns_false_when_global_vs_global() {
        let global1 = make_symbol(Binding::Global, true);
        let global2 = make_symbol(Binding::Global, true);
        assert!(!global1.is_stronger_than(&global2));
    }
}
```

シンボルの「強さ」のルールはシンプルで、`Local > Global`である。

- Local: ファイル内のみで有効。他のファイルから参照されない
- Global: 他のファイルから参照可能

:::message
ELFの規格上は`STB_WEAK`（弱シンボル）も存在し、`Local > Global > Weak`の3段階で
強さが定義されている。本書のサンプル（`main.o`と`sub.o`）にはWeakが登場しないので、
ここでは2段階に簡略化している。
:::

同じ名前のシンボルが複数ある場合、強い方が勝つ。

### is_stronger_thanを実装する

```diff:src/linker/output.rs
     pub fn is_stronger_than(&self, other: &Self) -> bool {
-        todo!()
+        match (self.info.binding, other.info.binding) {
+            // LOCALは最も強い
+            (Binding::Local, _) => true,
+            // 同じGLOBAL同士なら後勝ちしない
+            (Binding::Global, Binding::Global) => false,
+            // LOCALのほうがGLOBALより強い
+            (Binding::Global, Binding::Local) => false,
+        }
     }
```

:::message
厳密には`(Local, Local)`のケースもこの実装では`true`になる（`_`のワイルドカードで吸収される）。
ただし本書で扱う`main.o`と`sub.o`には同名のLOCALシンボルが他オブジェクト間で衝突する状況が出てこないので、簡略化である点に注意。
:::

### テストを実行する

```sh
$ cargo test linker::output
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running unittests src/lib.rs (target/debug/deps/tiny_linker-5558fbb2f7e5f511)

running 2 tests
test linker::output::tests::is_stronger_than_returns_false_when_global_vs_global ... ok
test linker::output::tests::is_stronger_than_returns_true_when_local_vs_global ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 7 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tiny_linker-bfb6a3022e853684)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## シンボル解決の実装

### テストを書く

ここでも先にテストを書く。
正常系（`main.o`の未定義`x`が`sub.o`の定義済み`x`で解決される）と、異常系（必要なシンボルが見つからない）の2つを用意する。

```rust:src/linker/symbol.rs
use std::collections::HashMap;

use crate::error::{LinkerError, Result};

use super::Linker;
use super::output::ResolvedSymbol;

impl Linker {
    pub fn resolve_symbols(&self) -> Result<HashMap<String, ResolvedSymbol>> {
        todo!()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::parser::Elf;

    #[test]
    fn resolve_symbols_returns_resolved_symbols() {
        let main_o = include_bytes!("../parser/fixtures/main.o");
        let sub_o = include_bytes!("../parser/fixtures/sub.o");

        let mut linker = Linker::default();
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
    fn resolve_symbols_returns_error_when_symbol_unresolved() {
        let main_o = include_bytes!("../parser/fixtures/main.o");

        let mut linker = Linker::default();
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

`main.o`と`sub.o`を例に、解決処理が何をしているかを順を追って見ていく。

main.oのシンボル：

- `_start`: GLOBAL, 定義済み（.textセクション）
- `x`: GLOBAL, 未定義（UND）

sub.oのシンボル：

- `x`: GLOBAL, 定義済み（.dataセクション）

これらを順に処理すると次のようになる。

1. `main.o`の`_start`を追加（定義済み）
2. `main.o`の`x`を追加（未定義）
3. `sub.o`の`x`を処理
   - 既存の`x`は未定義、新しい`x`は定義済み
   - 定義済みの方で上書きする → 解決成功

最終的に`_start`は`main.o`、`x`は`sub.o`で定義されている、という結果になる。

### resolve_symbolsを実装する

各オブジェクトのシンボルを順に見ていき、`HashMap`に詰めていく。
ぶつかったときは「既存と新規がそれぞれ定義済みか未定義か」「強さはどうか」で振り分ける、というのが基本的な流れである。

```diff:src/linker/symbol.rs
 use std::collections::HashMap;

+use crate::elf::symbol::SYMBOL_UNDEFINED;
 use crate::error::{LinkerError, Result};

 use super::output::ResolvedSymbol;
 use super::Linker;

 impl Linker {
     pub fn resolve_symbols(&self) -> Result<HashMap<String, ResolvedSymbol>> {
-        todo!()
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
+                    section_index: symbol.shndx,
+                    object_index: obj_idx,
+                    // section_indexがSYMBOL_UNDEFINED(0)でなければ定義済み
+                    is_defined: symbol.shndx != SYMBOL_UNDEFINED,
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
     }
 }
```

### テストを実行する

```sh
$ cargo test linker::symbol
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.00s
     Running unittests src/lib.rs (target/debug/deps/tiny_linker-5558fbb2f7e5f511)

running 2 tests
test linker::symbol::tests::resolve_symbols_returns_error_when_symbol_unresolved ... ok
test linker::symbol::tests::resolve_symbols_returns_resolved_symbols ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 9 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tiny_linker-bfb6a3022e853684)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## まとめ

本章ではシンボル解決を実装した。
シンプルなロジックだが、実際のリンカーの中核を担う処理であることを実感できたと思う。

- シンボルの強さは`Local > Global`の順で判定する
- 未定義シンボルは定義済みシンボルで上書きされる
- 同じ強さの定義済みシンボルが重複するとエラーにする
- 解決されないシンボルが残っていてもエラーにする

次章では、結合したセクションをメモリ上に配置していくセクション配置を実装していく。
