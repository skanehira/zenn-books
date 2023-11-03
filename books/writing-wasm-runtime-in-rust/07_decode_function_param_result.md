---
title: "関数の引数と戻り値のデコード実装"
---

前章ではもっともシンプルな関数のデコードを実装したが、以下のTODOを残していた。

- `Type Section`での引数と戻り値のデコード
- `Code Section`でのローカル変数と命令のデコード

本章ではそれらの処理を実装をしていく。

## 関数の引数と戻り値のデコード
改めて、前章でコンパイルしたWasmバイナリの`Type Section`は次のとおり。

```
; section "Type" (1)
0000008: 01               ; section code
0000009: 04               ; section size
000000a: 01               ; num types
; func type 0
000000b: 60               ; func
000000c: 00               ; num params
000000d: 00               ; num results
```

これを次の手順でデコードしていく。

1. 関数シグネチャの個数である`num types`を読み取る
    - この値の回数だけ2~6を繰り返す
2. `func`の値は読み捨てる
    - 関数シグネチャの種類を表す値だが、バージョン1では`0x60`固定
    - 本書では特に使わない
3. 引数の個数である`num params`を読み取る
4. 3で取得した値の数だけ、引数の型情報をデコードする
    - `0x7F`の場合は`i32`、`0x7E`の場合は`i64`なので、それぞれ`ValueType`に変換
5. 戻り値の個数である`num results`を読み取る
6. 5で取得した値の数だけ、戻り値の型情報をデコードする
    - `0x7F`の場合は`i32`、`0x7E`の場合は`i64`なので、それぞれ`ValueType`に変換

上記の処理を実装すると次のとおり。
コメントの番号がそれぞれ上記の手順である。

```diff:/src/binary/module.rs
 use super::{
     instruction::Instruction,
     section::{Function, SectionCode},
-    types::FuncType,
+    types::{FuncType, ValueType},
 };
 use nom::{
     bytes::complete::{tag, take},
+    multi::many0,
     number::complete::{le_u32, le_u8},
     sequence::pair,
     IResult,
@@ -93,10 +94,33 @@ fn decode_section_header(input: &[u8]) -> IResult<&[u8], (SectionCode, u32)> {
     ))
 }
 
-fn decode_type_section(_input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
-    let func_types = vec![FuncType::default()];
+fn decode_vaue_type(input: &[u8]) -> IResult<&[u8], ValueType> {
+    let (input, value_type) = le_u8(input)?;
+    Ok((input, value_type.into()))
+}
+
+fn decode_type_section(input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
+    let mut func_types: Vec<FuncType> = vec![];
+
+    let (mut input, count) = leb128_u32(input)?; // 1
+
+    for _ in 0..count {
+        let (rest, _) = le_u8(input)?; // 2
+        let mut func = FuncType::default();
+
+        let (rest, size) = leb128_u32(rest)?; // 3
+        let (rest, types) = take(size)(rest)?;
+        let (_, types) = many0(decode_vaue_type)(types)?; // 4
+        func.params = types;
+
+        let (rest, size) = leb128_u32(rest)?; // 5
+        let (rest, types) = take(size)(rest)?;
+        let (_, types) = many0(decode_vaue_type)(types)?; // 6
+        func.results = types;
 
-    // TODO: 引数と戻り値のデコード
+        func_types.push(func);
+        input = rest;
+    }
 
     Ok((&[], func_types))
 }
```

新しい関数`many0()`がでてきたので、それについて説明する。
`many0()`は受け取ったパーサーを使って、入力が終わるまでパースし続けて、入力の残りとパース結果を`Vec`で返す関数である。
これを使って、「`u8`を読み取って`ValueType`に変換」する関数である`decode_vaue_type()`を繰り返している。
このように、`many0()`を使うことで`for`を使ってデコードする必要がなくなり、実装がシンプルになる。

実装はできたので、続けてテストを実装していくが、現時点では引数のテストのみとする。
戻り値をテストするには命令のデコード処理を実装する必要があるので、次のセクションでまとめて実装する。

```diff:/src/binary/module.rs
 #[cfg(test)]
 mod tests {
     use crate::binary::{
-        instruction::Instruction, module::Module, section::Function, types::FuncType,
+        instruction::Instruction,
+        module::Module,
+        section::Function,
+        types::{FuncType, ValueType},
     };
     use anyhow::Result;
 
@@ -181,4 +184,26 @@ mod tests {
         );
         Ok(())
     }
+
+    #[test]
+    fn decode_func_param() -> Result<()> {
+        let wasm = wat::parse_str("(module (func (param i32 i64)))")?;
+        let module = Module::new(&wasm)?;
+        assert_eq!(
+            module,
+            Module {
+                type_section: Some(vec![FuncType {
+                    params: vec![ValueType::I32, ValueType::I64],
+                    results: vec![],
+                }]),
+                function_section: Some(vec![0]),
+                code_section: Some(vec![Function {
+                    locals: vec![],
+                    code: vec![Instruction::End],
+                }]),
+                ..Default::default()
+            }
+        );
+        Ok(())
+    }
 }
```

問題なければ次のとおり、テストが通る。

```sh
running 3 tests
test binary::module::tests::decode_simplest_module ... ok
test binary::module::tests::decode_func_param ... ok
test binary::module::tests::decode_simplest_func ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## ローカル変数と命令のデコード

前章でコンパイルしたWasmバイナリの`Code Section`は次のとおり。
`Code Section`は更に大きく分けて、ローカル変数の情報と命令コードがある。

```
; section "Code" (10)
0000012: 0a               ; section code
0000013: 04               ; section size
0000014: 01               ; num functions
; function body 0
0000015: 02               ; func body size
0000016: 00               ; local decl count
0000017: 0b               ; end
```

これを次の手順でデコードしていく。

1. 関数の個数である`num functions`を読み取る
    - この値の回数だけ2~5を繰り返す
2. `func body size`を読み取る
3. 2で取得した値のバイト列を切り出す
    - ローカル変数と命令のデコード処理の入力として使用する
4. 3で取得したバイト列を使ってローカル変数の情報をデコードする
5. 残りのバイト列を命令にデコードする


## まとめ
