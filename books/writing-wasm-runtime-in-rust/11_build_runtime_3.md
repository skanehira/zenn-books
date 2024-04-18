---
title: "Runtimeの実装 ~ \"Hello, World\"出力まで ~"
---

本章では前章までの実装を拡張して、以下の機能を実装していく。
これらの機能を実装すれば`"Hello, World"`を出力できるようになる。

- インポートした関数を実行する
- メモリのデータを標準出力に書き出す

## インポートした関数を実行する実装

Wasmバイナリの構造の章で以下のように説明をしたとおり、`Wasm Runtime`にはインポート機能がある。

> `Import Section`がモジュール外にあるメモリや関数などをインポートするための情報が定義されている領域である。
> モジュール外とは、他モジュールやRuntimeが用意したメモリや関数のことを指す。

本書では関数のインポートのみを実装する。
本節では最終的に以下のWATを実行できるようになる。

```wat:src/fixtures/import.wat
(module
  (func $add (import "env" "add") (param i32) (result i32))
  (func (export "call_add") (param i32) (result i32)
    (local.get 0)
    (call $add)
  )
)
```

### `Import Section`のデコード実装

`Import Section`のデコードは未実装なので、それを実装していく。
といっても`Export Section`の処理とほぼ同じなので、すんなり理解できると思う。

バイナリ構造は次とおり。

```
; section "Import" (2)
0000010: 02                ; section code
0000011: 0b                ; section size
0000012: 01                ; num imports
; import header 0
0000013: 03                ; string length
0000014: 656e 76           ; import module name (env)
0000017: 03                ; string length
0000018: 6164 64           ; import field name (add)
000001b: 00                ; import kind
000001c: 00                ; import signature index
```

まず、`Export`と同様に`Import`を追加する。

```diff:src/binary/types.rs
diff --git a/src/binary/types.rs b/src/binary/types.rs
index 191c34b..64912f8 100644
--- a/src/binary/types.rs
+++ b/src/binary/types.rs
@@ -36,3 +36,15 @@ pub struct Export {
     pub name: String,
     pub desc: ExportDesc,
 }
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub enum ImportDesc {
+    Func(u32),
+}
+
+#[derive(Debug, PartialEq, Eq)]
+pub struct Import {
+    pub module: String,
+    pub field: String,
+    pub desc: ImportDesc,
+}
```

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 5bf739d..dbcb30e 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -22,6 +22,7 @@ pub struct Module {
     pub function_section: Option<Vec<u32>>,
     pub code_section: Option<Vec<Function>>,
     pub export_section: Option<Vec<Export>>,
+    pub import_section: Option<Vec<Import>>,
 }
 
 impl Default for Module {
@@ -33,6 +34,7 @@ impl Default for Module {
             function_section: None,
             code_section: None,
             export_section: None,
+            import_section: None,
         }
     }
 }
```

次に、文字列をデコードする部分を`decode_name(...)`に切り出す。
バイナリを見たらわかるが、モジュール名と関数名は文字列なので、複数回文字列をデコードする必要があるので使い回せるようにする必要がある。

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index dbcb30e..7430fbe 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -214,9 +214,7 @@ fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
     let mut exports = vec![];
 
     for _ in 0..count {
-        let (rest, name_len) = leb128_u32(input)?;
-        let (rest, name_bytes) = take(name_len)(rest)?;
-        let name = String::from_utf8(name_bytes.to_vec()).expect("invalid utf-8 string");
+        let (rest, name) = decode_name(input)?;
         let (rest, export_kind) = le_u8(rest)?;
         let (rest, idx) = leb128_u32(rest)?;
         let desc = match export_kind {
@@ -230,6 +228,15 @@ fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
     Ok((input, exports))
 }
 
+fn decode_name(input: &[u8]) -> IResult<&[u8], String> {
+    let (input, size) = leb128_u32(input)?;
+    let (input, name) = take(size)(input)?;
+    Ok((
+        input,
+        String::from_utf8(name.to_vec()).expect("invalid utf-8 string"),
+    ))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::{
```

続けて、デコード処理を実装する。
やっていることは`Import Section`のデコードとほぼ同じである。


```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index 7430fbe..e3253de 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -2,7 +2,7 @@ use super::{
     instruction::Instruction,
     opcode::Opcode,
     section::{Function, SectionCode},
-    types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ValueType},
+    types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
@@ -83,6 +83,10 @@ impl Module {
                             let (_, exports) = decode_export_section(section_contents)?;
                             module.export_section = Some(exports);
                         }
+                        SectionCode::Import => {
+                            let (_, imports) = decode_import_section(section_contents)?;
+                            module.import_section = Some(imports);
+                        }
                         _ => todo!(),
                     };
 
@@ -228,6 +232,34 @@ fn decode_export_section(input: &[u8]) -> IResult<&[u8], Vec<Export>> {
     Ok((input, exports))
 }
 
+fn decode_import_section(input: &[u8]) -> IResult<&[u8], Vec<Import>> {
+    let (mut input, count) = leb128_u32(input)?;
+    let mut imports = vec![];
+
+    for _ in 0..count {
+        let (rest, module) = decode_name(input)?;
+        let (rest, field) = decode_name(rest)?;
+        let (rest, import_kind) = le_u8(rest)?;
+        let (rest, desc) = match import_kind {
+            0x00 => {
+                let (rest, idx) = leb128_u32(rest)?;
+                (rest, ImportDesc::Func(idx))
+            }
+            _ => unimplemented!("unsupported import kind: {:X}", import_kind),
+        };
+
+        imports.push(Import {
+            module,
+            field,
+            desc,
+        });
+
+        input = rest;
+    }
+
+    Ok((&[], imports))
+}
+
 fn decode_name(input: &[u8]) -> IResult<&[u8], String> {
     let (input, size) = leb128_u32(input)?;
     let (input, name) = take(size)(input)?;
```

続けてテストコードを追加して、デコードの実装が問題ないことを確認する。

```diff:src/binary/module.rs
diff --git a/src/binary/module.rs b/src/binary/module.rs
index e3253de..2a8dbfa 100644
--- a/src/binary/module.rs
+++ b/src/binary/module.rs
@@ -275,7 +275,7 @@ mod tests {
         instruction::Instruction,
         module::Module,
         section::Function,
-        types::{Export, ExportDesc, FuncType, FunctionLocal, ValueType},
+        types::{Export, ExportDesc, FuncType, FunctionLocal, Import, ImportDesc, ValueType},
     };
     use anyhow::Result;
 
@@ -427,4 +427,39 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_import() -> Result<()> {
+        let wasm = wat::parse_file("src/fixtures/import.wat")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![ValueType::I32],
+                    results: vec![ValueType::I32],
+                }]),
+                import_section: Some(vec![Import {
+                    module: "env".into(),
+                    field: "add".into(),
+                    desc: ImportDesc::Func(0),
+                }]),
+                export_section: Some(vec![Export {
+                    name: "call_add".into(),
+                    desc: ExportDesc::Func(1),
+                }]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![],
+                    code: vec![
+                        Instruction::LocalGet(0),
+                        Instruction::Call(0),
+                        Instruction::End
+                    ],
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

```sh
running 10 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok
test binary::module::tests::decode_func_local ... ok
test binary::module::tests::decode_func_call ... ok
test binary::module::tests::decode_func_add ... ok
test execution::runtime::tests::not_found_export_function ... ok
test binary::module::tests::decode_import ... ok
test execution::runtime::tests::func_call ... ok
test execution::runtime::tests::execute_i32_add ... ok
```

### インポート関数実行の実装

## メモリのデータを標準出力に書き出す実装

