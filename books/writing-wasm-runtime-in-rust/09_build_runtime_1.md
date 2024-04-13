---
title: "Runtimeの実装 その1"
---

本章では次のWATを実行できるようRuntimeを実装していく。
実装できた段階では簡単な足し算ができる`Wasm Runtime`が出来上がる。

```wat
(module
  (func (param i32 i32) (result i32)
    (local.get 0)
    (local.get 1)
    i32.add
  )
)
```

処理の流れは大きく分けると次のようになる。

1. `binary::Module`を使って`execution::Store`を生成する
2. `execution::Store`を使って`execution::Runtime`を生成する
3. `execution::Runtime::call(...)`で関数を実行する

詳細は実装する際に解説する。

## 値の実装

今回実装するWasm Runtimeでは次の2種類の値を扱うのでそれらを実装していく。

- i32
- i64

まずは`src`配下に次のファイルを作成する。

- `src/execution/value.rs`
- `src/execution.rs`

それぞれ次のように実装する。

```rust:src/execution/value.rs
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Value {
    I32(i32),
    I64(i64),
}

impl From<i32> for Value {
    fn from(value: i32) -> Self {
        Value::I32(value)
    }
}

impl From<i64> for Value {
    fn from(value: i64) -> Self {
        Value::I64(value)
    }
}

impl std::ops::Add for Value {
    type Output = Self;
    fn add(self, rhs: Self) -> Self::Output {
        match (self, rhs) {
            (Value::I32(left), Value::I32(right)) => Value::I32(left + right),
            (Value::I64(left), Value::I64(right)) => Value::I64(left + right),
            _ => panic!("type mismatch"),
        }
    }
}
```

```rust:src/execution.rs
pub mod value;
```

```diff:src/execution/lib.rs
diff --git a/src/lib.rs b/src/lib.rs
index 96eab66..ec63376 100644
--- a/src/lib.rs
+++ b/src/lib.rs
@@ -1 +1,2 @@
 pub mod binary;
+pub mod execution;
```

`i32`と`i64`は異なる型なので、スタックでまとめて扱うことができない。
そのため`Value`というEnumを用意してスタックで扱えるようしている。

また`Value`同士を足し算できるように`std::ops::Add`を実装している。

## `Store`の実装

`Store`[^1]は`Wasm Runtime`が実行時に持つ状態を保持するための構造体である。
仕様書ではメモリやインポート、関数などの情報を持つと定義されていて、これらの情報を使って命令の処理をしていく。
`Wasm`バイナリがクラスだとしたら、`Store`はそのクラスのインスタンスと考えることができる。

現時点では関数の情報を持っていれば良いので、次のファイルを作成して`Store`を実装する。

```rust:src/execution/store.rs
use crate::binary::{
    instruction::Instruction,
    types::{FuncType, ValueType},
};

#[derive(Clone)]
pub struct Func {
    pub locals: Vec<ValueType>,
    pub body: Vec<Instruction>,
}

#[derive(Clone)]
pub struct InternalFuncInst {
    pub func_type: FuncType,
    pub code: Func,
}

#[derive(Clone)]
pub enum FuncInst {
    Internal(InternalFuncInst),
}

#[derive(Default)]
pub struct Store {
    pub funcs: Vec<FuncInst>,
}
```

`FuncInst`は`Wasm Runtime`が実際に処理する関数の実体となっている。
関数はインポートした関数とWasmバイナリが持つ関数がある。
今回はインポートした関数を使わないので、まずWasmバイナリが持つ関数を表現した`InternalFuncInst`を実装する。

`InternalFuncInst`のフィールドはそれぞれ次のようになっている

- `func_type`: 関数のシグネチャ（引数と戻り値）情報
- `code`: 関数のローカル変数の定義と命令列を持つ`Func`型

次に、`binary::Module`を受け取り`Store`を生成する関数を実装する。

```diff:src/execution/store.rs
diff --git a/src/execution/store.rs b/src/execution/store.rs
index e488383..5b4e467 100644
--- a/src/execution/store.rs
+++ b/src/execution/store.rs
@@ -1,7 +1,9 @@
 use crate::binary::{
     instruction::Instruction,
+    module::Module,
     types::{FuncType, ValueType},
 };
+use anyhow::{bail, Result};
 
 #[derive(Clone)]
 pub struct Func {
@@ -23,3 +25,44 @@ pub enum FuncInst {
 pub struct Store {
     pub funcs: Vec<FuncInst>,
 }
+
+impl Store {
+    pub fn new(module: Module) -> Result<Self> {
+        let func_type_idxs = match module.function_section {
+            Some(ref idxs) => idxs.clone(),
+            _ => vec![],
+        };
+
+        let mut funcs = vec![];
+
+        if let Some(ref code_section) = module.code_section {
+            for (func_body, type_idx) in code_section.iter().zip(func_type_idxs.into_iter()) {
+                let Some(ref func_types) = module.type_section else {
+                    bail!("not found type_section")
+                };
+
+                let Some(func_type) = func_types.get(type_idx as usize) else {
+                    bail!("not found func type in type_section")
+                };
+
+                let mut locals = Vec::with_capacity(func_body.locals.len());
+                for local in func_body.locals.iter() {
+                    for _ in 0..local.type_count {
+                        locals.push(local.value_type.clone());
+                    }
+                }
+
+                let func = FuncInst::Internal(InternalFuncInst {
+                    func_type: func_type.clone(),
+                    code: Func {
+                        locals,
+                        body: func_body.code.clone(),
+                    },
+                });
+                funcs.push(func);
+            }
+        }
+
+        Ok(Self { funcs })
+    }
+}
```

実装で使用する各種セクションはそれぞれ次のデータを持っている。


| セクション         | 概要                                 |
|--------------------|--------------------------------------|
| `Type Section`     | 関数シグネチャの情報                 |
| `Code Section`     | 関数ごとの命令などの情報             |
| `Function Section` | 関数シグネチャへの参照情報           |

現時点`Store::new()`でやっていることは簡潔に説明すると、各セクションに散らばっている関数を実行する際に必要な情報を集めている。
少し分かりづらいと思うので、次の`func_add.wat`の`binary::Module`をみていこう。

```rust
Module {
    type_section: Some(vec![FuncType {
        params: vec![ValueType::I32, ValueType::I32],
        results: vec![ValueType::I32],
    }]),
    function_section: Some(vec![0]),
    code_section: Some(vec![Function {
        locals: vec![],
        code: vec![
            Instruction::LocalGet(0),
            Instruction::LocalGet(1),
            Instruction::I32Add,
            Instruction::End
        ],
    }]),
    ..Default::default()
}
```

`code_section[0]`の関数がどんなシグネチャを持っているかを知るには`function_section[0]`の値を取得する必要がある。
その値は`type_section`のインデックスになっているので、そのまま`type_section[0]`が`code_section[0]`のシグネチャ情報になる。
例えば`function_section[0]`の値が1だった場合、`type_section[1]`が`code_section[0]`のシグネチャ情報になる。

## Rntimeの実装


[^1]: https://www.w3.org/TR/wasm-core-1/#store%E2%91%A0
