---
title: "Runtimeの実装 その2"
---

本章では前章までの実装を拡張して、以下の機能を実装していく。

- エクスポートした関数のみを実行する
- インポートした関数を実行する
- 関数から関数を呼び出す
- メモリのデータを標準出力に書き出す

これらの機能を実装すれば`Hello World`を出力できるようになる。

## エクスポートした関数のみを実行する

前章ではインデックスで実行する関数を指定していた。

```rust
let args = vec![Value::I32(left), Value::I32(right)];
let result = runtime.call(0, args)?;
assert_eq!(result, Some(Value::I32(want)));
```

これではバイナリ構造を解析しないと実行したい関数を指定できないので非常に使いづらい。
また、`Wasm spec`ではエクスポートした関数のみを実行できるという仕様があるが、今の実装は仕様を満たしていない。
そのため、本節では関数名を指定して関数を実行できるようにする。

### `Export Section`のデコード実装
`func_add.wat`を次のように修正して、`add`という関数名で関数をエクスポートする。

```diff:src/fixtures/func_add.wat
diff --git a/src/fixtures/func_add.wat b/src/fixtures/func_add.wat
index ce14757..99678c4 100644
--- a/src/fixtures/func_add.wat
+++ b/src/fixtures/func_add.wat
@@ -1,5 +1,5 @@
 (module
-  (func (param i32 i32) (result i32)
+  (func (export "add") (param i32 i32) (result i32) 
     (local.get 0)
     (local.get 1)
     i32.add
```

`Export Section`のデコードを実装していないため現時点ではテストが失敗するので、それを実装する。
バイナリ構造に関してはすでに[Wasmの入門](https://zenn.dev/skanehira/books/writing-wasm-runtime-in-rust/viewer/04_wasm_binary_structure)で解説したのでそちらを参照してほしい。

まずエクスポートを表現した型を定義する。

```diff:src/binary/types.rs
diff --git a/src/binary/types.rs b/src/binary/types.rs
index 7707a97..191c34b 100644
--- a/src/binary/types.rs
+++ b/src/binary/types.rs
@@ -25,3 +25,14 @@ pub struct FunctionLocal {
     pub type_count: u32,
     pub value_type: ValueType,
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub enum ExportDesc {
+    Func(u32),
+}
+
+#[derive(Debug, PartialEq, Eq)]
+pub struct Export {
+    pub name: String,
+    pub desc: ExportDesc,
+}
```

`Export::name`はエクスポート名で今回は`add`になる。
`ExportDesc`は関数やメモリなどの実体への参照となっていて、関数の場合は`Store::funcs`のインデックス値になる。
たとえば`ExportDesc::Func(0)`の場合、`add`という名前の関数の実態は`Store::funcs[0]`になる。
本書ではメモリのエクスポートなどは実装しないので、`ExportDesc::Func`のみを実装する。

続けて`Export Section`のデコードを実装する。

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 5ea23a1..949aea9 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{FuncType, FunctionLocal, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -194,6 +194,27 @@ fn decode_instructions(input: &[u8]) -> IResult<&[u8], Instruction> {
     Ok((rest, inst))
 }
 
+fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
+    let (mut input, count) = leb128_u32(input)?; // 1
+    let mut exports = vec![];
+
+    for _ in 0..count { // 9
+        let (rest, name_len) = leb128_u32(input)?; // 2
+        let (rest, name_bytes) = take(name_len)(rest)?; // 3
+        let name = String::from_utf8(name_bytes.to_vec()).expect("invalid utf-8 string"); // 4
+        let (rest, export_kind) = le_u8(rest)?; // 5
+        let (rest, idx) = leb128_u32(rest)?; // 6
+        let desc = match export_kind { // 7
+            0x00 => ExportDesc::Func(idx),
+            _ => unimplemented!("unsupported export kind: {:X}", export_kind),
+        };
+        exports.push(Export { name, desc }); // 8
+        input = rest;
+    }
+
+    Ok((input, exports))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::{
```

`decode_export_section(...)`では次のことをやっている。

1. エクスポートの要素数を取得
   今回は関数1つしかエクスポートしていないので1になる
2. エクスポート名のバイト列の長さを取得
3. 2で取得した長さの分だけバイト列を取得
4. 3で取得したバイト列を文字列に変換する
5. エクスポートの種類（関数、メモリなど）を取得
6. エクスポートした関数など実体への参照を取得
7. `0x00`の場合、エクスポートの種類は関数なので`Export`を生成
8. 配列に7で生成した`Export`を追加
9. 2から8を要素数の分だけ繰り返す

これで`Export Section`のデコードを実装できたので、続けて、`Module`に`export_section`を追加する。

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index d14704f..41c9a89 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{FuncType, FunctionLocal, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -21,6 +21,7 @@ pub struct Module {
     pub type_section: Option<Vec<FuncType>>,
     pub function_section: Option<Vec<u32>>,
     pub code_section: Option<Vec<Function>>,
+    pub export_section: Option<Vec<Export>>,
 }
 
 impl Default for Module {
@@ -31,6 +32,7 @@ impl Default for Module {
             type_section: None,
             function_section: None,
             code_section: None,
+            export_section: None,
         }
     }
 }
@@ -72,6 +74,10 @@ impl Module {
                             let (_, funcs) = decode_code_section(section_contents)?;
                             module.code_section = Some(funcs);
                         }
+                        SectionCode::Export => {
+                            let (_, exports) = decode_export_section(section_contents)?;
+                            module.export_section = Some(exports);
+                        }
                         _ => todo!(),
                     };
 
@@ -221,7 +227,7 @@ mod tests {
         instruction::Instruction,
         module::Module,
         section::Function,
-        types::{FuncType, FunctionLocal, ValueType},
+        types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
     };
     use anyhow::Result;
```

テストも修正する。

```diff:src/
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 41c9a89..9ba5afc 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -329,6 +329,10 @@ mod tests {
                         Instruction::End
                     ],
                 }]),
+                export_section: Some(vec![Export {
+                    name: "add".into(),
+                    desc: ExportDesc::Func(0),
+                }]),
                 ..Default::default()
             }
         );
```

これでテストも通るようになる。

```sh
running 6 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_param ... ok
test execution::runtime::tests::execute_i32_add ... ok
test binary::module::tests::decode_func_add ... ok
```

### 実行する実装

## インポーポした関数を実行する

## 関数から関数を呼び出す

## メモリのデータを標準出力に書き出す

## まとめ
