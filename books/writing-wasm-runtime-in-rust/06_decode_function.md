---
title: "関数のデコード実装"
---

前章ではデコードの基本的な実装方法について解説をしたので、本章では関数をデコードできるように実装していく。

まず、手始めに次のようなもっとも小さな関数をデコードできるように実装をしていく。
```wat
(module
  (func)
)
```

上記のWATをWasmバイナリにコンパイルすると次のようなバイナリ構造になる。

```
0000000: 0061 736d        ; WASM_BINARY_MAGIC
0000004: 0100 0000        ; WASM_BINARY_VERSION
; section "Type" (1)
0000008: 01               ; section code
0000009: 04               ; section size
000000a: 01               ; num types
; func type 0
000000b: 60               ; func
000000c: 00               ; num params
000000d: 00               ; num results
; section "Function" (3)
000000e: 03               ; section code
000000f: 02               ; section size
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
; section "Code" (10)
0000012: 0a               ; section code
0000013: 04               ; section size
0000014: 01               ; num functions
; function body 0
0000015: 02               ; func body size (guess)
0000016: 00               ; local decl count
0000017: 0b               ; end
```

## セクションのデコード
関数のデコードを実装するには、次のセクションのデコード処理を実装する必要がある。

| セクション         | 概要                                 |
|--------------------|--------------------------------------|
| `Type Section`     | 関数シグネチャの情報                 |
| `Code Section`     | 関数ごとの命令などの情報             |
| `Function Section` | 関数シグネチャへの参照情報           |

各種セクションのフォーマットについては[Wasmバイナリの構造の章](/books/writing-wasm-runtime-in-rust/04_wasm_binary_structure%252Emd)で説明したとおりなので、それを参照してもらいながら実装について解説していく。

### セクションヘッダーのデコード
各種セクションには必ず`section code`と`section size`を持つセクションヘッダーがあるので、
まずは`src/binary/section.rs`ファイルを作成して、`section code`のEnumを定義する。

```rust:src/binary/section.rs
use num_derive::FromPrimitive;

#[derive(Debug, PartialEq, Eq, FromPrimitive)]
pub enum SectionCode {
    Custom = 0x00,
    Type = 0x01,
    Import = 0x02,
    Function = 0x03,
    Memory = 0x05,
    Global = 0x06,
    Export = 0x07,
    Start = 0x08,
    Element = 0x09,
    Code = 0x0a,
    Data = 0x0b,
}
```

```diff:src/binary.rs
 pub mod module;
+pub mod section;
```

この`SectionCode`をみて、各セクションのデコード処理が分岐していくことになる。

```rust
match code {
    SectionCode::Type => {
        ...
    }
    SectionCode::Function => {
        ...
    }
    ...
}
```

`SectionCode`を用意できたので、次にセクションヘッダーをデコードする関数`decode_section_header()`を実装していく。
この関数は入力から`section code`と`section size`を取得するだけだが、いくつか新しいことをやっているのでそれについて解説していく。

```diff:src/binary/module.rs
-use nom::{IResult, number::complete::le_u32, bytes::complete::tag};
+use super::section::SectionCode;
+use nom::{
+    bytes::complete::tag,
+    number::complete::{le_u32, le_u8},
+    sequence::pair,
+    IResult,
+};
+use nom_leb128::leb128_u32;
+use num_traits::FromPrimitive as _;
 
 #[derive(Debug, PartialEq, Eq)]
 pub struct Module {
@@ -34,6 +42,17 @@ impl Module {
     }
 }
 
+fn decode_section_header(input: &[u8]) -> IResult<&[u8], (SectionCode, u32)> {
+    let (input, (code, size)) = pair(le_u8, leb128_u32)(input)?; // ①
+    Ok((
+        input,
+        (
+            SectionCode::from_u8(code).expect("unexpected section code"), // ②
+            size,
+        ),
+    ))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

①の`pair`は複数のパーサーをもとに、新しいパーサーを返すパーサーである。
`section code`と`section size`はフォーマットが決まっているので、それぞれをパースする関数をまとめて1回の関数呼び出しで処理できるように`pair`を使っている。
`pair`を使わない場合は、次のような実装になる。

```rust
let (input, code) = le_u8(input);
let (input, size) = leb128_u32(input);
```

`section code`は1バイト固定なので今回は`le_u8()`を使っている。
`section size`はLEB128[^1]でエンコードされた`u32`なので、値を読み取る際は`leb128_u32()`を使う必要がある。

:::message alert
`Wasm spec`では数値はすべてLEB128でエンコードすると定められていることに注意
筆者はこれを知らず、テスト通らなくて数時間を溶かしたことがある
:::

②の`SectionCode::from_u8()`は`num_derive::FromPrimitive`マクロで実装された関数である。
読み取った1バイトの数値から`SectionCode`に変換するために使っている。
これを使わない場合、次のように筋肉で解決する必要が出てくる。

```rust
impl From<u8> for SectionCode {
    fn from(code: u8) -> Self {
        match code {
            0x00 => Self::Custom,
            0x01 => Self::Type,
            ...
        }
    }
}
```

[^1]: 任意の大きさなの数値を少ないバイト数で格納するための可変長符号圧縮の方式

セクションヘッダーのデコードの実装はできたので、続けて次のようにデコード処理の骨組みを実装していく。

```diff:src/binary/module.rs
 use super::section::SectionCode;
 use nom::{
-    bytes::complete::tag,
+    bytes::complete::{tag, take},
     number::complete::{le_u32, le_u8},
     sequence::pair,
     IResult,
@@ -38,6 +38,29 @@ impl Module {
             magic: "\0asm".into(),
             version,
         };
+
+        let mut remaining = input;
+
+        while !remaining.is_empty() { // ①
+            match decode_section_header(remaining) { // ②
+                Ok((input, (code, size))) => {
+                    let (rest, section_contents) = take(size)(input)?; // ③
+
+                    match code {
+                        _ => todo!(), // ④
+                    };
+
+                    remaining = rest; // ④
+                }
+                Err(err) => return Err(err),
+            }
+        }
         Ok((input, module))
     }
 }
```

上記の実装では、次のことをやっている。

- ① 入力が空になるまで、②~⑤の処理をループする
- ② セクションヘッダーをデコードし、セクションコードとサイズ、残りの入力を取得
- ③ セクションサイズ分のバイト列を残りの入力から更に取得する（`take()`は指定したサイズ分だけ読み取るパーサー）
- ④ 各種セクションのデコード処理（これから実装していく）
- ⑤ 残りの入力を次のループで使うため、`remaining`に再代入

やっていることはシンプルだが、これを読んでピンと来ない方も多いと思う。
なので、バイナリ構造をもとに上記の②~⑤の処理を考えてみる。

まず`Type Section`とその次の`Function Section`のバイナリ構造体の部分を抜き出すと次のとおりである。

```
; section "Type" (1)
0000008: 01               ; section code
0000009: 04               ; section size
000000a: 01               ; num types
; func type 0
000000b: 60               ; func
000000c: 00               ; num params
000000d: 00               ; num results
; section "Function" (3)
000000e: 03               ; section code
000000f: 02               ; section size
0000010: 01               ; num functions
0000011: 00               ; function 0 signature index
...
```

①の時点では`remaining`は次のようになっている。

| remaining                                                       |
|-----------------------------------------------------------------|
| [0x01, 0x04, 0x01, 0x60, 0x0, 0x0, 0x03, 0x02, 0x01, 0x00, ...] |


②が終わった時点で、`input`などが次のようになっている。

| section code | section size | input                                               |
|--------------|--------------|-----------------------------------------------------|
| 0x01         | 0x04         | [0x01, 0x60, 0x0, 0x0, 0x03, 0x02, 0x01, 0x00, ...] |

③が終わった時点で`rest`と`section_contents`は次のようになっている。

| section_contents       | rest                          |
|------------------------|-------------------------------|
| [0x01, 0x60, 0x0, 0x0] | [0x03, 0x02, 0x01, 0x00, ...] |

④では`section_contents`を更にデコードしていく。

⑤では`remaining`に`rest`の値が入る、この時点で次のセクションのデータ入力としてまた②に戻って処理を行う

| remaining                     |
|-------------------------------|
| [0x03, 0x02, 0x01, 0x00, ...] |

このように、繰り返し入力を消費して各セクションをデコード処理していく。
理解してしまえばシンプルだが、理解するまでは繰り返し読み直したり、読者自身で書いてみたりすると良いと思う。

### `Type Section`のデコード
骨組みができたので、続けて`Type Section`のデコード処理を実装していく。
`Type Section`は関数シグネチャ情報を持つセクションで、シグネチャは引数と戻り値の組み合わせである。
そのため、まずは`src/binary/types.rs`ファイルを作って、シグネチャの構造体を定義していく。

```rust:src/binary/types.rs
#[derive(Debug, Default, Clone, PartialEq, Eq)]
pub struct FuncType {
    pub params: Vec<ValueType>,
    pub results: Vec<ValueType>,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ValueType {
    I32, // 0x7F
    I64, // 0x7E
}

impl From<u8> for ValueType {
    fn from(value: u8) -> Self {
        match value {
            0x7F => Self::I32,
            0x7E => Self::I64,
            _ => panic!("invalid value type: {:X}", value),
        }
    }
}
```

```diff:src/binary.rs
 pub mod module;
 pub mod section;
+pub mod types;
```

:::message
`Wasm spec`では`i32`、`i64`、`f32`、`f64`の4つの値が定義されているが、今回は`i32`と`i64`があれば良いので`ValueType`はその2つのみ実装する
:::

続けて、`Module`構造体に`type_section`フィールドを追加するのと、④の`todo!()`を実装していく。

```diff:src/binary/module.rs
-use super::section::SectionCode;
+use super::{section::SectionCode, types::FuncType};
 use nom::{
     bytes::complete::{tag, take},
     number::complete::{le_u32, le_u8},
@@ -12,6 +12,7 @@ use num_traits::FromPrimitive as _;
 pub struct Module {
     pub magic: String,
     pub version: u32,
+    pub type_section: Option<Vec<FuncType>>,
 }
 
 impl Default for Module {
@@ -19,6 +20,7 @@ impl Default for Module {
         Self {
             magic: "\0asm".to_string(),
             version: 1,
+            type_section: None,
         }
     }
 }
@@ -34,9 +36,10 @@ impl Module {
         let (input, _) = tag(b"\0asm")(input)?;
         let (input, version) = le_u32(input)?;
 
-        let module = Module {
+        let mut module = Module {
             magic: "\0asm".into(),
             version,
+            ..Default::default()
         };
 
         let mut remaining = input;
@@ -47,6 +50,10 @@ impl Module {
                     let (rest, section_contents) = take(size)(input)?;
 
                     match code {
+                        SectionCode::Type => {
+                            let (_, types) = decode_type_section(section_contents)?;
+                            module.type_section = Some(types);
+                        }
                         _ => todo!(),
                     };
 
```

`decode_type_section()`は実際に`Type Sectio`のデコード処理をしている関数だが、今回は引数も戻り値もない最もシンプルな関数なのでひとまず何もしなくても問題ない。
引数と戻り値のデコードは別途実装する。

```diff:src/binary/module.rs
@@ -77,6 +77,14 @@ fn decode_section_header(input: &[u8]) -> IResult<&[u8], (SectionCode, u32)> {
     ))
 }
 
+fn decode_type_section(_input: &[u8]) -> IResult<&[u8], Vec<FuncType>> {
+    let func_types = vec![];
+
+    // TODO: 引数と戻り値のデコード
+
+    Ok((&[], func_types))
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

### `Function Section`のデコード

### `Code Section`のデコード
