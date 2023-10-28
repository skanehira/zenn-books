---
title: "Wasmバイナリのデコード"
---

本章では[nom](https://crates.io/crates/nom)という[パーサーコンビネーター](https://en.wikipedia.org/wiki/Parser_combinator)を使ってWasmバイナリをデコードしていく。
パーサーコンビネーターを使わなくても実装はできるが、実装量が減るので本書は`nom`を採用することにした。

## 準備
早速Rustのプロジェクトを作成して、必要なクレートを導入しよう。

```sh
$ cargo new tiny-wasm-runtime --name tinywasm
```

プロジェクトを作成したら、次のクレートを`Cargo.toml`に以下を追記する。

```toml:Cargo.toml
[dependencies]
anyhow = "1.0.71"
nom = "7.1.3"
nom-leb128 = "0.2.0"
num-derive = "0.4.0"
num-traits = "0.2.15"

[dev-dependencies]
wat = "=1.0.67"
pretty_assertions = "1.4.0"
```

ちなみに本章のプロジェクトレイアウトは最終的に次のようになる。

```
tiny-wasm-runtime/
 src/
  binary/
   instruction.rs # Wasm命令の定義
   module.rs      # モジュールの定義
   opcode.rs      # オペコードの定義
   section.rs     # 各種セクションの定義
   types.rs       # 各種型の定義
   fixtures/      # 各種テストデータ
    call.wat
    data.wat
    export.wat
   ...
  binary.rs
  execution.rs
  lib.rs
  main.rs
```

## プリアンブルのデコード
プリアンブルは前章で説明したとおり、次のようなバイナリ構造になっている。

```
           \0asm
         ┌───┴───┐
0000000: 0061 736d      ; WASM_BINARY_MAGIC
~~~~~~~  ~~             ~~~~~~~~~~~~~~~~~~~~ 
 │        │                   │
 │        │                   └ コメント
 │        └ 16進数表記、2桁で1バイト
 └ アドレスのオフセット

0000004: 0100 0000      ; WASM_BINARY_VERSION
```

これをRustの構造体で表現すると、次のとおり。とてもシンプルなのが分かる。

```rust
pub struct Module {
    pub magic: String,
    pub version: u32,
}
```

データ構造がわかったので、次は実装を進めていく。
実装は小さく初めるのがよいので、最初にもっとも小さなモジュールを定義して、そのデコード処理を実装していく。

まずは`src`配下に次のファイルを作成する。

- `src/binary.rs`
- `src/lib.rs`
- `src/binary/module.rs`

それぞれ、次のように記述する

```rust:src/binary/module.rs
#[derive(Debug, PartialEq, Eq)]
pub struct Module {
    pub magic: String,
    pub version: u32,
}
```

```rust:src/binary.rs
pub mod module;
```

```rust:src/lib.rs
pub mod binary;
```

次にテストを実装していく。
テストは基本的にWATで書いたコードをWasmバイナリにコンパイルして、それをデコードした結果が想定したとおりに構造体に値が入っているかを確認していく。

```diff:src/binary/module.rs
@@ -3,3 +3,17 @@ pub struct Module {
     pub magic: String,
     pub version: u32,
 }
+
+#[cfg(test)]
+mod tests {
+    use crate::binary::module::Module;
+    use anyhow::Result;
+
+    #[test]
+    fn decode_simplest_module() -> Result<()> {
+        // 最も小さなモジュールを定義して、wasmバイナリにコンパイル
+        let wasm = wat::parse_str("(module)")?;
+        // バイナリをデコードしてModule構造体を生成
+        let module = Module::new(&wasm)?;
+        // 生成したModule構造体が想定通りになっているかを比較
+        assert_eq!(module, Module::default());
+        Ok(())
+    }
+}
```

上記のコードで分かるように、テストをパスするためには`Moduele::new()`と`Module::default()`を実装する必要がある。
まずは`Default()`を実装していく。

```diff:src/binary/module.rs
     pub version: u32,
 }
 
+impl Default for Module {
+    fn default() -> Self {
+        Self {
+            magic: "\0asm".to_string(),
+            version: 1,
+        }
+    }
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

`Magic number`とバージョンは不変なので、`Default`トレイトを実装して記述量を減らす。

続けて、デコード処理の実装をしていく。

```diff:src/binary/module.rs
+use nom::{IResult, number::complete::le_u32, bytes::complete::tag};
+
 #[derive(Debug, PartialEq, Eq)]
 pub struct Module {
     pub magic: String,
@@ -13,6 +15,25 @@ impl Default for Module {
     }
 }
 
+impl Module {
+    pub fn new(input: &[u8]) -> anyhow::Result<Module> {
+        let (_, module) =
+            Module::decode(input).map_err(|e| anyhow::anyhow!("failed to parse wasm: {}", e))?;
+        Ok(module)
+    }
+
+    fn decode(input: &[u8]) -> IResult<&[u8], Module> {
+        let (input, _) = tag(b"\0asm")(input)?;
+        let (input, version) = le_u32(input)?;
+
+        let module = Module {
+            magic: "\0asm".into(),
+            version,
+        };
+        Ok((input, module))
+    }
+}
+
 #[cfg(test)]
 mod tests {
     use crate::binary::module::Module;
```

これでプリアンブルのデコード処理が実装できたので、テストを実行してみると通ることが分かる。

```sh
$ cargo test decode_simplest_module
    Finished test [unoptimized + debuginfo] target(s) in 0.05s
     Running unittests src/lib.rs (target/debug/deps/tinywasm-010073c10c93afeb)

running 1 test
test binary::module::tests::decode_simplest_module ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/tinywasm-9670d80381f93079)
```

### デコード処理の解説
`nom`を使ったことがない方は上記のコードを読んでもよくわからないと思うのですこし解説する。
特にわからない方は読み飛ばして問題ない。

まず、`nom`は入力を受け取って、読み取った部分と残りの部分をタプルで返すという設計になっている。
なので、たとえば次の`le_u32()`に入力を渡すと、`(残りの部分, 読み取った部分)`という感じの結果を得られる。

```rust
let (input, version) = le_u32(input)?;
```

`le_u32()`は`nom`が提供しているパーサーの1つで、リトルエンディアンで4バイト読み取った値を`u32`に変換した結果を返してくれる。
なので、バイト列から`u32`な数値を取得したい場合はこの関数を使えばよい。

また、`nom`は`tag()`というパーサーも提供していている。
こちらは入力を読み取り、渡した値と一致しない場合はエラーを返すという挙動をする。
つまり入力のバリデーションと読み取りを同時に処理できるともいえる。

上記のコードを見ると`b"\0asm"`を`tag()`に渡して、入力を読み取って残りの入力だけを取得する処理ということが分かる。

```rust
let (input, _) = tag(b"\0asm")(input)?;
```

ちなみに、渡した値と入力が一致しない場合は次のようなエラーが発生する。

```sh
Error: failed to parse wasm: Parsing Error: Error { input: [0, 97, 115, 109, 1, 0, 0, 0], code: Tag }
```

一度まとめると、`decode()`関数の処理は

- バイナリの先頭から4バイトを読み取り、`\0asm`であれば残りの入力を受取る
- 残りの入力からさらに4バイト読み取って残りと`u32`に変換した値の入力を受取る

ということをやっている。

バイナリのデコードは基本的にこのようなことを繰り返していくだけなので、やること自体はとてもシンプルである。
最初は中々慣れないと思うが、繰り返し書いてみると慣れてくると思うので、焦らずにゆっくりやっていくとよいと思う。

ちなみに筆者も最初は慣れなくて少し戸惑っていたが、慣れてくると結構直感的に使えるなという印象を持っている。

## セクションのデコード

### セクションヘッダーのデコード

### 関数をデコード

関数をデコードできるようには、次のセクションのデコード処理を実装する必要がある。

- Type Section
- Function Section
- Code Section
