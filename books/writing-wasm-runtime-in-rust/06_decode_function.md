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

この`SectionCode`をみて、デコード処理が分岐していくことになる。

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
`section code`と`section size`はフォーマットが決まっているので、それぞれをパースする関数まとめて1回の関数呼び出しで処理できるように`pair`を使っている。
`pair`を使わない場合は、次のような実装になる。

```rust
let (input, code) = le_u8(input);
let (input, size) = leb128_u32(input);
```

`section code`は1バイト固定なので今回は`le_u8()`を使っている。
`section size`はLEB128[^1]でエンコードされた`u32`なので、値を読み取る際は`leb128_u32()`を使う必要がある。

:::message alert
`Wasm spec`では数値はすべてLEB128でエンコードすると定められているので要注意
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
