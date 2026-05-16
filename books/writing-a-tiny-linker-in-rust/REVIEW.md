# REVIEW: 『Rustで小さなリンカーを実装する』改善提案

## はじめに

このドキュメントは、本書『Rustで小さなリンカーを実装する』を
**リンカ未経験の読者がそのままなぞって完走でき、かつ理解の腹落ちが残る本** に
するための改善提案をまとめたものである。

### 検証経緯
本書の各章を別ディレクトリ (`/Users/skanehira/dev/github.com/skanehira/tiny-linker-book/`)
にゼロから書き起こし、`docker exec linker-develop` (`rust:1.95-slim-bullseye` / arm64) で
全章の `cargo test`（16 件）と E2E 検証（`./a.out; echo $? = 11`）を再現した。
その過程で検出した「読者が詰まる箇所」を整理した。

### 優先度の凡例
- 🔴 **カテゴリA / 動作を阻む誤り**（8 件）— 直さないと読者がそのままなぞれない／章間で矛盾する
- 🟡 **カテゴリB / 軽微な改善**（6 件）— 動くが、コード品質や読みやすさに影響
- 🟢 **カテゴリC / 説明補強**（6 件）— 写経はできるが「なぜ」が薄い箇所への加筆

### このドキュメントの使い方
1. サマリ表で全体像を把握
2. カテゴリA → C → B の順に各項目を確認
3. 採用する提案だけを手動で対応する `.md` に適用
4. 適用後、本書末尾の「適用後の検証手順」を実行して回帰しないことを確認

### 配置上の注意
このファイル (`REVIEW.md`) は本書ディレクトリ内に置いているが、`config.yaml` の
`chapters:` リストに含めていないため Zenn 上の章としては公開されない。社内レビュー専用。

---

## サマリ表

| #   | 優先度 | 章                            | 行                    | 概要                                                               | 適用難易度             |
| --- | ------ | ----------------------------- | --------------------- | ------------------------------------------------------------------ | ---------------------- |
| A-1 | 🔴     | 01_intro                      | 12-21                 | `__asm__(...)` 末尾の `;` 抜け                                     | 易（1文字）            |
| A-2 | 🔴     | 13_impl_executable            | 176-185               | 同上、動作確認用 main.c の `;` 抜け                                | 易（1文字）            |
| A-3 | 🔴     | 02_intro_linker               | 178-183               | `readelf -S a.out` の数値が標準 `ld` と矛盾                        | 易（数値差し替え）     |
| A-4 | 🔴     | 03_elf_binary_structure       | 151-171               | セクション `[4]` `[5]` が無言で欠落                                | 易（省略記号）         |
| A-5 | 🔴     | 03_elf_binary_structure       | 348                   | `.rela.text` offset `0x178` → 実機 `0x180`                         | 易（1値）              |
| A-6 | 🔴     | 03_elf_binary_structure       | 403-416               | `readelf -l a.out` の Entry / count / FileSiz が実機と乖離         | 中（ブロック差し替え） |
| A-7 | 🔴     | 08_how_linker_works           | 118                   | 助詞ミス「ベースアドレスは `0x400000` について、」                 | 易                     |
| A-8 | 🔴     | 08_how_linker_works           | 27-53                 | 擬似コード `parser::parse_elf(&input)?.1` が実 API と乖離          | 中（コード差し替え）   |
| B-1 | 🟡     | 05_impl_parser_header         | 60-66                 | ディレクトリ図の `linker/mod.rs` を `linker.rs` に統一             | 易                     |
| B-2 | 🟡     | 05_impl_parser_header         | 307, 456              | `ELF_IDENT_HEADER_SIZE` の名前と用途のズレ                         | 易（命名変更）         |
| B-3 | 🟡     | 06_impl_parser_section_symbol | 307-314               | `Binding` に `Weak` が無い／注記なし                               | 易（注記追加）         |
| B-4 | 🟡     | 09_impl_symbol_resolution     | 174-186               | `is_stronger_than` の `(Local, Local) → true` 注記なし             | 易（注記追加）         |
| B-5 | 🟡     | 10_impl_section_layout        | 455-465               | `shndx` を `value >= 0x410000` で判定する hack                     | 中（実装変更）         |
| B-6 | 🟡     | 10, 11_impl_relocation        | 10章: 478 / 11章: 258 | `apply_relocations` の引数型が `&mut Vec<T>` ↔ `&mut [T]` で揺れる | 易（型統一）           |
| C-1 | 🟢     | 08_how_linker_works           | 117 付近              | `.text` オフセットが `0x100` になる理由（算術）の追加              | 中（節追加）           |
| C-2 | 🟢     | 10_impl_section_layout        | 211 付近              | `text_offsets / data_offsets` の役割とシンボル更新式の図示         | 中（節追加）           |
| C-3 | 🟢     | 11_impl_relocation            | 346 付近              | `0x9F00001F` マスクの導出説明                                      | 易（2-3 文追加）       |
| C-4 | 🟢     | 12_impl_elf_output            | 333 付近              | `.symtab` の `sh_link` / `sh_info` / `sh_entsize` 規約解説         | 中（節追加）           |
| C-5 | 🟢     | 09_impl_symbol_resolution     | 165 付近              | 「実物のリンカには `Weak` もある」の注記                           | 易（1-2 文）           |
| C-6 | 🟢     | 13_impl_executable            | 末尾                  | 「次に読む参考資料」リンク集                                       | 易（節追加）           |

---

## 🔴 カテゴリA: 動作を阻む誤り

### A-1. ch01 / 行 12-21: `__asm__(...)` 末尾の `;` 抜け

**現状** (`01_intro.md` 12-21):

```c:main.c
__asm__(
      ".global _start\n"
      "_start:\n"
      "    adr     x0, x\n"
      "    ldr     w0, [x0]\n"
      "    mov     x8, #93\n"
      "    svc     #0\n"
)
```

**提案**:

```c:main.c
__asm__(
      ".global _start\n"
      "_start:\n"
      "    adr     x0, x\n"
      "    ldr     w0, [x0]\n"
      "    mov     x8, #93\n"
      "    svc     #0\n"
);
```

**理由**: 末尾 `);` の `;` が欠落しており、このまま `gcc -c main.c` するとコンパイルエラー
（`error: expected ';' before '}' token`）になる。02章 (`02_intro_linker.md` 103-112行) の
同じスニペットでは正しく `);` が書かれているので、01章だけが脱字。
読者は本書の冒頭で「ゴール確認のためにこのコードを試そう」と動かしうるため、優先度高。

---

### A-2. ch13 / 行 176-185: 動作確認用 main.c の `;` 抜け（A-1 と同じ）

**現状** (`13_impl_executable.md` 176-185):

```sh
$ cat > main.c << 'EOF'
__asm__(
      ".global _start\n"
      "_start:\n"
      "    adr     x0, x\n"
      "    ldr     w0, [x0]\n"
      "    mov     x8, #93\n"
      "    svc     #0\n"
)
EOF
```

**提案**:

```sh
$ cat > main.c << 'EOF'
__asm__(
      ".global _start\n"
      "_start:\n"
      "    adr     x0, x\n"
      "    ldr     w0, [x0]\n"
      "    mov     x8, #93\n"
      "    svc     #0\n"
);
EOF
```

**理由**: A-1 と同じ問題。最終章で読者がコピペで `main.c` を作る箇所なので、ここが落ちると
本書全体のゴールである `exit 11` の再現に失敗する。

---

### A-3. ch02 / 行 178-183: `readelf -S a.out` の数値が標準 `ld` と矛盾

**現状** (`02_intro_linker.md` 178-183):

```sh
$ readelf -S a.out | grep -E "Nr|\.text|\.data"
  [Nr] Name              Type             Address           Offset
  [ 1] .text             PROGBITS         0000000000400100  00000100
  [ 2] .data             PROGBITS         0000000000410110  00000110
```

**提案**:

```sh
$ readelf -S a.out | grep -E "Nr|\.text|\.data"
  [Nr] Name              Type             Address           Offset
  [ 1] .text             PROGBITS         00000000004000e8  000000e8
  [ 2] .data             PROGBITS         00000000004100f8  000000f8
```

**理由**: 02章は「標準 `ld -o a.out main.o sub.o` で a.out を作って `readelf` で覗く」という文脈。
実機（gcc 10.x / ld 2.35 / arm64-linux-gnu）の標準 `ld` 出力は `.text=0x4000e8 / .data=0x4100f8`。
本書の現状値 `0x400100 / 0x410110` は **本書の自作リンカが生成する値**で、ここに混ぜると章内で
矛盾する（同じ章 232-234行のまとめ図は `.text → 0x4000e8 / .data → 0x4100f8` で実機と整合）。
標準 `ld` の値に揃えるのが筋。

---

### A-4. ch03 / 行 151-171: セクション `[4] .comment` `[5] .note.GNU-stack` が無言で欠落

**現状** (`03_elf_binary_structure.md` 151-171):

```sh
$ readelf -S sub.o
There are 9 section headers, starting at offset 0x1a0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000000  0000000000000000  AX       0     0     1
  [ 2] .data             PROGBITS         0000000000000000  00000040
       0000000000000004  0000000000000000  WA       0     0     4
  [ 3] .bss              NOBITS           0000000000000000  00000044
       0000000000000000  0000000000000000  WA       0     0     1
  [ 6] .symtab           SYMTAB           0000000000000000  00000070
       00000000000000d8  0000000000000018           7     8     8
  [ 7] .strtab           STRTAB           0000000000000000  00000148
       000000000000000c  0000000000000000           0     0     1
  [ 8] .shstrtab         STRTAB           0000000000000000  00000154
       0000000000000045  0000000000000000           0     0     1
```

**提案**: `[3]` の直後に省略記号と一文を挿入し、本書のスコープ外であることを明示する。

```sh
$ readelf -S sub.o
There are 9 section headers, starting at offset 0x1a0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000000  0000000000000000  AX       0     0     1
  [ 2] .data             PROGBITS         0000000000000000  00000040
       0000000000000004  0000000000000000  WA       0     0     4
  [ 3] .bss              NOBITS           0000000000000000  00000044
       0000000000000000  0000000000000000  WA       0     0     1
  ...  # [4] .comment / [5] .note.GNU-stack は本書では扱わないため省略
  [ 6] .symtab           SYMTAB           0000000000000000  00000070
       00000000000000d8  0000000000000018           7     8     8
  [ 7] .strtab           STRTAB           0000000000000000  00000148
       000000000000000c  0000000000000000           0     0     1
  [ 8] .shstrtab         STRTAB           0000000000000000  00000154
       0000000000000045  0000000000000000           0     0     1
```

**理由**: `[3]` の次にいきなり `[6]` が来るので、読者が「`[4]` と `[5]` はどこ？」と立ち止まる。
省略記号 `...` で意図的に省いていることを示す。実機の `sub.o` には `[4] .comment` `[5] .note.GNU-stack`
が並ぶ。

---

### A-5. ch03 / 行 348: `.rela.text` の offset `0x178` → 実機 `0x180`

**現状** (`03_elf_binary_structure.md` 348):

```
Relocation section '.rela.text' at offset 0x178 contains 1 entry:
```

**提案**:

```
Relocation section '.rela.text' at offset 0x180 contains 1 entry:
```

**理由**: 同じ `main.o` に対して、02章 195-198行は `at offset 0x180`、03章だけが `0x178`。
実機（gcc 10.2.1 / arm64）は `0x180`。`0x180` に統一する。

---

### A-6. ch03 / 行 403-416: `readelf -l a.out` の値が標準 `ld` の実機と乖離

**現状** (`03_elf_binary_structure.md` 403-416):

```sh
$ readelf -l a.out
Elf file type is EXEC (Executable file)
Entry point 0x400078
There are 2 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000088 0x0000000000000088  R E    0x10000
  LOAD           0x0000000000000088 0x0000000000410088 0x0000000000410088
                 0x0000000000000004 0x0000000000000004  RW     0x10000
```

**提案**:

```sh
$ readelf -l a.out
Elf file type is EXEC (Executable file)
Entry point 0x4000e8
There are 3 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000000f8 0x00000000000000f8  R E    0x10000
  LOAD           0x00000000000000f8 0x00000000004100f8 0x00000000004100f8
                 0x0000000000000004 0x0000000000000004  RW     0x10000
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
```

**理由**: 03章は標準 `ld` で作った `a.out` を題材にしている。実機の標準 `ld` は次の値を出す:
- エントリ `0x4000e8`
- プログラムヘッダ **3 個**（LOAD / LOAD / **GNU_STACK**）
- LOAD#1 の `FileSiz = 0xf8`
- LOAD#2 の `Offset / VirtAddr = 0xf8 / 0x4100f8`

本書の値はどの組み合わせとも一致せず、読者が `readelf` を叩いた瞬間に立ち止まる。

---

### A-7. ch08 / 行 118: 助詞ミス

**現状** (`08_how_linker_works.md` 118):

```
ベースアドレスは`0x400000`について、GNU `ld`がAArch64 Linux向けにデフォルトで使うベースアドレスがこの値なので、本書の自作リンカーもそれに倣っている。
```

**提案**:

```
ベースアドレス`0x400000`は、GNU `ld`がAArch64 Linux向けにデフォルトで使う値なので、本書の自作リンカーもそれに倣っている。
```

**理由**: 「〜は〜について、〜なので」は日本語として破綻している。

---

### A-8. ch08 / 行 27-53: 擬似コードの API 名が実 API と乖離

**現状** (`08_how_linker_works.md` 27-53):

```rust
pub fn link_to_file(&mut self, inputs: Vec<Vec<u8>>) -> Result<Vec<u8>, Error> {
    // 1. 入力ファイルをパース
    for input in inputs {
        let obj = parser::parse_elf(&input)?.1;
        self.objects.push(obj);
    }

    // 2. シンボル解決
    let mut resolved_symbols = self.resolve_symbols()?;
    ...
}
```

**提案**: 07章で実装した `Elf::parse` の API に揃え、13章の最終コードに直結する形にする。

```rust
pub fn link_to_file(&mut self, inputs: Vec<Vec<u8>>) -> Result<Vec<u8>> {
    // 1. 入力ファイルをパース
    for (idx, input) in inputs.iter().enumerate() {
        let obj = Elf::parse(input).map_err(|e| LinkerError::Parse(format!("{:?}", e)))?;
        self.add_object(format!("input_{}", idx), obj);
    }

    // 2. シンボル解決
    let mut resolved_symbols = self.resolve_symbols()?;

    // 3. セクション配置
    let (output_sections, section_name_offsets) =
        self.layout_sections(&mut resolved_symbols)?;

    // 4. 実行ファイルの書き出し
    let mut out = std::io::Cursor::new(Vec::new());
    self.write_executable(
        &mut out,
        resolved_symbols,
        output_sections,
        section_name_offsets,
    )?;

    Ok(out.into_inner())
}
```

**理由**: 本書 07章で実装した API は `Elf::parse(raw) -> Result<Elf, ParseError>`（`nom` の
`(rest, value)` を返す関数ではない）。`parser::parse_elf(&input)?.1` のような関数は本書中で
一度も登場しない。「ざっくり」と前置きしてあるが、章をまたいだ API の連続性が断たれて読者が
混乱する。13章の `link_to_file` 完成形 (`13_impl_executable.md` 76-101行) と読み比べると、
このコードがほぼそのまま完成形と分かる構成にしたい。

---

## 🟡 カテゴリB: 軽微な改善

### B-1. ch05 / 行 60-66: ディレクトリ図の `linker/mod.rs` を `linker.rs` に統一

**現状** (`05_impl_parser_header.md` 60-66):

```
└── linker/
    ├── mod.rs
    ├── symbol.rs
    ├── section.rs
    ├── relocation.rs
    ├── output.rs
    └── writer.rs
```

**提案**:

```
├── linker.rs
└── linker/
    ├── symbol.rs
    ├── section.rs
    ├── relocation.rs
    ├── output.rs
    └── writer.rs
```

**理由**: 本書の他章（09, 10, 11, 12, 13章）の実装は `src/linker.rs` 形式（Rust 2018+ 標準）で
書かれている。05章の図だけ Rust 2015 の `mod.rs` 形式で書かれており、章間で表記が割れる。

---

### B-2. ch05 / 行 307, 456: `ELF_IDENT_HEADER_SIZE` の名前と用途のズレ

**現状** (`05_impl_parser_header.md` 307 / 456):

```rust
const ELF_IDENT_HEADER_SIZE: usize = 16;
...
pub fn parse(raw: &[u8]) -> ParseResult<'_, Header> {
    if raw.len() < ELF_IDENT_HEADER_SIZE {
        return Err(nom::Err::Error(ParseError::InvalidHeaderSize(raw.len() as u8)));
    }
    ...
}
```

**提案**: 定数名を `e_ident` 部分の長さと明確にする。

```rust
const ELF_IDENT_SIZE: usize = 16;
```

**理由**: 定数名 `ELF_IDENT_HEADER_SIZE = 16` は「`e_ident` 部分（16B）」を指しているように
読めるが、用途は「ELF ヘッダ全体（64B）のサイズチェック」。名前と用途がねじれている。
`ELF_IDENT_SIZE` に改名すれば「`e_ident` (16B) が読めるだけのバイト数があるか先に確認している」
という意図のままになり、残りの長さは `nom` がパースエラーで拾う、という現状の挙動と整合する。

---

### B-3. ch06 / 行 307-314: `Binding` enum に `Weak` がない

**現状** (`06_impl_parser_section_symbol.md` 307-314):

```rust
pub enum Binding {
    #[default]
    Local = 0,
    Global = 1,
}
```

**提案**: コードはそのままにし、enum 定義の直後（314行付近）に注記を入れる:

```markdown
> 03章で紹介した `STB_WEAK`（弱シンボル）は本書のスコープ外なので enum に含めていない。
> 標準 C ライブラリのリンクなどを扱うと必要になるが、本書のサンプルでは登場しないため省略する。
```

**理由**: 03章 317-321行で `STB_WEAK = 2` を表で紹介済みなのに、06章の `Binding` enum には
含まれていない。これに気づいた読者は「漏れ？」と感じる。意図的な省略であることを明示する。

---

### B-4. ch09 / 行 174-186: `is_stronger_than` の `(Local, Local) → true` 注記

**現状** (`09_impl_symbol_resolution.md` 174-186):

```rust
pub fn is_stronger_than(&self, other: &Self) -> bool {
    match (self.info.binding, other.info.binding) {
        // LOCALは最も強い
        (Binding::Local, _) => true,
        // 同じGLOBAL同士なら後勝ちしない
        (Binding::Global, Binding::Global) => false,
        // LOCALのほうがGLOBALより強い
        (Binding::Global, Binding::Local) => false,
    }
}
```

**提案**: コードはそのままにし、186行の `}` 直後に次の注記を加える:

```markdown
> 厳密には `(Local, Local)` のケースもこの実装では `true` になる（`_` のワイルドカードで吸収される）。
> ただし本書で扱う `main.o` と `sub.o` には同名の LOCAL シンボルが他オブジェクト間で衝突する状況が
> 出てこないので、テストでは表面化しない。実用リンカでは異なる翻訳単位の同名 LOCAL は別物として
> 扱う（共存可能）ため、ここはあくまで簡略化である点に注意。
```

**理由**: コードのロジックを読み飛ばさずに追う読者にとって、`(Local, Local) → true` が
「両方ローカルなら新しい方で上書きしていいの？」という疑問を残す。本書のスコープでは
問題にならないことを明示する。

---

### B-5. ch10 / 行 455-465: `shndx` を `value >= 0x410000` で判定する hack

**現状** (`10_impl_section_layout.md` 455-465):

```rust
// st_shndx (2バイト) - セクションインデックス（1=.text, 2=.data）
let shndx = if symbol.section_index == 0 {
    0u16
} else {
    // 簡略化: .textを1, .dataを2とする
    if symbol.value >= 0x410000 {
        2u16
    } else {
        1u16
    }
};
```

**提案**: `ResolvedSymbol` に「最終的にどの出力セクションに属すか」を持たせるか、
`merge_sections` の中で確定したセクション割り当てを `shndx` に書き戻す形にする。
最小差分案として、`value` 比較を `text_section.addr` / `data_section.addr` ベースに変える:

```rust
// st_shndx (2バイト) - 1=.text, 2=.data
let shndx = if symbol.section_index == 0 {
    0u16
} else if symbol.value >= /* data_section.addr */ 0x410000 {
    2u16
} else {
    1u16
};
```

加えて本文で「アドレス値で振り分けているのは簡略化。複数 `.text` セクションや `.rodata`
などを足す段階で破綻するので、その時は `ResolvedSymbol` 側に出力セクションのインデックスを
持たせるよう拡張する」と明示する。

**理由**: アドレス値で `shndx` を決めるロジックは、`.rodata` や複数 `.data` を扱った瞬間に
壊れる。本書のスコープでは動くが、拡張するつもりの読者がここで足を取られる。
コードを大幅変更すると章バランスが崩れるので、注記で「これは簡略化」を伝えるだけでもよい。

---

### B-6. ch10 / 行 478 と ch11 / 行 258: `apply_relocations` の引数型を統一

**現状**:

`10_impl_section_layout.md` 476-482:
```rust
pub fn apply_relocations(
    &self,
    _sections: &mut Vec<Section<'static>>,
    _resolved_symbols: &HashMap<String, ResolvedSymbol>,
) -> Result<()> {
    Ok(())
}
```

`11_impl_relocation.md` 行 258 付近:
```rust
pub fn apply_relocations(
    &self,
    output_sections: &mut [Section<'static>],
    resolved_symbols: &HashMap<String, ResolvedSymbol>,
) -> Result<()> {
```

**提案**: 10章の stub も `&mut [Section<'static>]` で統一する:

```rust
pub fn apply_relocations(
    &self,
    _sections: &mut [Section<'static>],
    _resolved_symbols: &HashMap<String, ResolvedSymbol>,
) -> Result<()> {
    Ok(())
}
```

**理由**: 10章で `&mut Vec<T>` で書いた stub を、11章で **「スタブを削除する」** と説明しつつ
`&mut [T]` で書き直している。型シグネチャの変化が「次章で実装」と「スタブ削除」の挙動の
混在で見えにくい。Rust の deref coercion で動くが、本書を写経する読者にとっては
シグネチャが変わったことを忘れる。最初から `&mut [T]` で揃えるのが素直。

---

## 🟢 カテゴリC: 説明補強

### C-1. ch08: `.text` オフセットが `0x100` になる理由

**挿入位置**: `08_how_linker_works.md` の「メモリレイアウト」節（行 96-116）と
「ベースアドレス `0x400000`」の説明（行 117-120）の **間**。新規見出し
`### .text オフセット 0x100 の内訳` として挿入。

**目的**: ch10 のコード `let text_offset = 0x100;` がどこから出てくるかは、
ch10 のコード内コメント 1 行で済まされているが、本来は ch08 の地ならしで説明すべき重要な数値。
ELFヘッダ＋プログラムヘッダ＋パディングを実数で見せる。

**文案**:

```markdown
### .text オフセット 0x100 の内訳

`.text` をファイル先頭から `0x100` バイト目に置くのには、次のような理由がある。

ELFヘッダーは固定で 64 バイト (`0x40`)、プログラムヘッダーは 1 個あたり 56 バイト (`0x38`)。
本書では `.text` 用と `.data` 用に LOAD セグメントを 2 個作るので、両者を合わせると次のようになる。

| 領域 | サイズ | 終端オフセット |
| --- | --- | --- |
| ELF Header | `0x40` | `0x40` |
| Program Header × 2 | `0x38 × 2 = 0x70` | `0xb0` |
| **パディング** | `0x50` | `0x100` |
| .text | (コードサイズ) | `0x100 + len(.text)` |

`.text` の直前に `0x50` バイトのパディングを入れているのは、`0x100` ＝ 256 バイトという
「キリのいい」境界に揃えるためで、特別な規約があるわけではない。`0xb0` から始めても
動くが、後で章をまたいでアドレス計算する際に `0x100` のほうが暗算しやすい。

このオフセット `0x100` はベースアドレス `0x400000` と合わさって、`.text` の仮想アドレス
`0x400100` を決定する。これが 13章で生成する `a.out` の `Entry point: 0x400100` の正体である。
```

---

### C-2. ch10: `text_offsets` / `data_offsets` の役割とシンボル更新式

**挿入位置**: `10_impl_section_layout.md` 行 211 の小見出し
`### セクション配置を実装する` の直後にある説明文（「やることは大きく3つで〜」、行 212-213）の
**直後**、テストコードの前。新規小節 `### マージ後オフセットの記録` として挿入。

**目的**: コード内で突然出てくる `text_offsets: HashMap<(usize, u16), usize>` というデータ構造
の意義を、絵と式で先に提示しておく。後段の「シンボルアドレス更新の仕組み」(498-516行) と
連動して読みやすくなる。

**文案**:

```markdown
### マージ後オフセットの記録

複数のオブジェクトファイルから `.text` を集めて 1 本にマージするとき、
**「元の (オブジェクト, セクション) がマージ後の何バイト目に置かれたか」** を覚えておく必要がある。
そうしないと、シンボルが「自分のもとのセクション内オフセット」しか持っていないので、
マージ後の絶対アドレスに変換できない。

たとえば次のような状況を考える。

```
main.o の .text (16 バイト) ─┐
                              ├─ マージ後の .text (20 バイト)
sub.o  の .text  (4 バイト) ─┘
```

このとき:

- `main.o の .text` はマージ後 `0` バイト目から
- `sub.o の .text` はマージ後 `16` バイト目から

これを覚えておくために、本書では次のような `HashMap` を使う。

```rust
// キー: (オブジェクトのインデックス, 元のセクションインデックス)
// 値:   マージ後の .text 内オフセット
let mut text_offsets: HashMap<(usize, u16), usize> = HashMap::new();
```

シンボルの最終アドレスは、この情報を使って次のように決まる:

```
シンボルの最終アドレス
  = .text セクションのベースアドレス
  + マージ後の (オブジェクト, セクション) オフセット
  + シンボルが持っていた元のセクション内オフセット
```

たとえば `sub.o` の `.text` 内オフセット 2 にあるシンボルなら、
`0x400100 (.text の addr) + 16 (sub.o の .text マージ後オフセット) + 2 (元オフセット) = 0x400112`
が最終アドレスになる。

`.data` についても同じ仕組みで `data_offsets` を作る。
```

---

### C-3. ch11: `0x9F00001F` マスクの導出

**挿入位置**: `11_impl_relocation.md` の「再配置処理を実装する」内、`let opcode_rd = instruction & 0x9F00001F;`
の前後（行 346 付近）。差分ブロック内のコメントに 2-3 文を足すか、ブロック直前の本文に橋を入れる。

**目的**: ADR 命令のビットレイアウト図（行 98-110）と `0x9F00001F` の間に飛びがあり、
読者が「なぜこのマスク？」で止まる。32-bit の各ビット位置を実数で繋ぐ。

**文案** (差分ブロックの直前に挿入):

```markdown
ここで `0x9F00001F` というマスクが出てくるが、これは ADR 命令の中で
**「即値以外の部分」を残すためのマスク** である。ビット位置で表すと:

```
ビット位置: 31 30 29 28 27 26 25 24 23 ... 5 4 3 2 1 0
内訳:        op immlo immlo 1  0  0  0  0  immhi(19bit) Rd Rd Rd Rd Rd
マスク:       1  0  0  1  1  1  1  1  0...0  1  1  1  1  1
                                       ↑ ここを 0 にして即値を消す
```

二進で `1001_1111_0000_0000_0000_0000_0001_1111` ＝ 16進で `0x9F00001F`。
これでオペコード（ビット31）、固定パターン（ビット28-24）、レジスタ番号（ビット4-0）だけを
残し、即値部分（`immlo` ビット30-29、`immhi` ビット23-5）をゼロクリアできる。
あとは新しい `immlo / immhi` を OR で書き戻せば、書き換え後の命令が組み上がる。
```

---

### C-4. ch12: `.symtab` の `sh_link` / `sh_info` / `sh_entsize` 規約解説

**挿入位置**: `12_impl_elf_output.md` の `write_executable` 実装の中で、`.symtab` だけ
特別扱いしている箇所（行 333-353 付近）の **直前** に、新規見出し
`### .symtab セクションヘッダの規約` として挿入。

**目的**: 突然「`.symtab` のときは `sh_link` に `.strtab` のインデックス、`sh_info` に LOCAL の個数
を入れる」というコードが出てくる。ELF 仕様（System V gABI）の規約であることを明示する。

**文案**:

```markdown
### .symtab セクションヘッダの規約

`.symtab` セクションのヘッダだけは、他のセクションと違って `sh_link` / `sh_info` / `sh_entsize`
の各フィールドに **意味のある値** を入れる必要がある。System V gABI で次のように規定されている。

| フィールド | 意味 | 本書での値 |
| --- | --- | --- |
| `sh_link` | シンボル名を引く文字列テーブルセクションのインデックス | `.strtab` の section header index |
| `sh_info` | 最後の LOCAL シンボルのインデックス + 1（= GLOBAL の開始位置） | LOCAL シンボル数 + 1（先頭の NULL 分） |
| `sh_entsize` | 1 エントリのサイズ | ELF64 では 24 |

`sh_info` の「LOCAL の個数 + 1」になるのは、シンボルテーブルの先頭に必ず NULL シンボルを
1 個置く規約があるため。つまりインデックス `[0]` が NULL、`[1..sh_info]` が LOCAL、
`[sh_info..]` が GLOBAL という並びになる。

`readelf -s` で `.symtab` を表示するときも、リンカや動的ローダがシンボルを引くときも、
この `sh_link` から `.strtab` を辿ってシンボル名を取得する。これらを正しくセットしないと
`readelf -s` で `<corrupt: ...>` のような表示になるので、出力後の検証で気づきやすい。
```

---

### C-5. ch09: 「実物のリンカには Weak もある」の注記

**挿入位置**: `09_impl_symbol_resolution.md` の「シンボルの「強さ」のルールはシンプルで、
`Local > Global`である。」（行 165）の **直後**、箇条書きの後（行 169 の `同じ名前のシンボル〜`
の前）に追記。

**目的**: 03章で `STB_WEAK = 2` を紹介したのに、09章では `Weak` が出てこない。
読者が「Weak は？」と思った時に答える注記を 1 行入れる。

**文案**:

```markdown
> ELF の規格上は `STB_WEAK`（弱シンボル）も存在し、`Local > Global > Weak` の 3 段階で
> 強さが定義されている。本書のサンプル（`main.o` と `sub.o`）には Weak が登場しないので、
> ここでは 2 段階に簡略化している。
```

---

### C-6. ch13: まとめ末尾の書籍紹介を独立セクションに格上げ

**挿入位置**: `13_impl_executable.md` のまとめ末尾、現状 273-276 行で本文の続きとして
紹介されている書籍リンクを、独立した見出し `## さらに学ぶための参考資料` に切り出す。

**目的**: 現状は「まとめ」の地の文に書籍リンクが埋もれており、本書を完走した読者の
「次の一歩」として目に留まりにくい。独立節として見出しを立てるだけで存在感が増す。
紹介する書籍は筆者自身が読んだ 1 冊（Amazon 本）に絞り、未読のリソースは載せない。

**文案**:

```markdown
## さらに学ぶための参考資料

本書では最小限の静的リンクの実装にとどめた。動的リンクや PLT/GOT、最適化など、
リンカ周りをより深く学びたい読者には、筆者も参照した次の書籍を勧める。

- [リンカ・ローダ実用技術](https://www.amazon.co.jp/dp/4789838072)
  — リンカ・ローダの仕組みと実装を網羅的に扱った定番書。本書の内容を含めて、
  より踏み込んだ内容を学びたい読者に最適。
```

---

## 適用順序の提案

1. **ステップ 1（🔴 カテゴリA）**: A-1 → A-2 → A-3 → A-4 → A-5 → A-6 → A-7 → A-8 の順に適用。
   読者の完走を阻む箇所から潰す。
2. **ステップ 2（🟢 カテゴリC）**: C-1 → C-2 → C-3 → C-4 → C-5 → C-6 の順。理解の腹落ち補強。
   本書の章順 (08 → 09 → 10 → 11 → 12 → 13) に沿って書いてある。
3. **ステップ 3（🟡 カテゴリB）**: B-1 → B-2 → B-3 → B-4 → B-5 → B-6 の順。コード品質と
   注記の追加。動作には影響しないので最後でよい。

各ステップ完了後に、次節の検証手順を回して回帰がないことを確認する。

---

## 適用後の検証手順

修正した本書 `.md` を「読者が初めて読む状態」と仮定して、`tiny-linker-book/` を一度
ゼロから作り直し、cargo test と E2E が壊れていないことを確認する。

### 1. 検証用ディレクトリを作り直す

```sh
rm -rf /tmp/verify-tiny-linker-book
mkdir /tmp/verify-tiny-linker-book
cd /tmp/verify-tiny-linker-book
git init
# 以下、本書 02章から順に compose.yaml / main.c / sub.c を作成し、
# 05〜13 章のコードを本書のとおりに貼っていく
```

### 2. 各章で `cargo test`

| 章 | 期待される結果                                                                                |
| -- | --------------------------------------------------------------------------------------------- |
| 05 | 2 件パス（`parse_elf_header`, `invalid_magic_number`）                                        |
| 06 | 4 件パス（+ `parse_section_headers`, `parse_symbols`）                                        |
| 07 | 6 件パス（+ `parse_relocations`, `parse_elf`）                                                |
| 09 | 10 件パス（+ `is_stronger_than_*` 2 件, `resolve_symbols_*` 2 件）                            |
| 10 | 12 件パス（+ `align_*`, `layout_sections_*`）                                                 |
| 11 | 13 件パス（+ `apply_relocations`）                                                            |
| 12 | 15 件パス（+ `header_to_bytes_returns_64_bytes`, `write_executable_writes_valid_elf_header`） |
| 13 | 16 件パス（+ `link_to_file_returns_valid_elf`）                                               |

### 3. 13章末の E2E

```sh
docker compose exec linker-develop bash -c '
  gcc -c main.c -o main.o
  gcc -c sub.c -o sub.o
  cargo run --release -- a.out main.o sub.o
  ./a.out; echo $?
'
# 期待: Linked successfully: a.out / 11
```

### 4. 章間の数値整合性チェック

修正後の本書を全章 grep して、次の数値が **すべて同じ値で揃っている** ことを確認:

```sh
grep -n '0x400100\|0x400e8\|0x4000e8' books/writing-a-tiny-linker-in-rust/*.md
grep -n '0x410110\|0x410088\|0x4100f8' books/writing-a-tiny-linker-in-rust/*.md
grep -n '0x178\|0x180' books/writing-a-tiny-linker-in-rust/*.md
```

期待:
- 標準 `ld` 出力（02章, 03章）: `.text=0x4000e8 / .data=0x4100f8 / .rela.text=0x180`
- 本書リンカ出力（08, 13章）: `.text=0x400100 / .data=0x410110`
- 各章のまとめ図とコード例で値が混在しないこと

### 5. readelf 出力例の実機照合

修正した readelf 出力例（A-3, A-5, A-6）は、コンテナ内で実機の出力と完全一致することを確認:

```sh
docker compose exec linker-develop bash -c '
  gcc -c main.c -o main.o && gcc -c sub.c -o sub.o
  ld -o a.out main.o sub.o
  readelf -S a.out | grep -E "\.text|\.data"
  readelf -l a.out
  readelf -r main.o
'
```

---

## 全 20 件チェックリスト

- [x] A-1: ch01 `__asm__()` → `__asm__();`
- [x] A-2: ch13 `__asm__()` → `__asm__();`
- [x] A-3: ch02 readelf -S a.out を `0x4000e8 / 0x4100f8` に
- [x] A-4: ch03 セクション省略記号を入れる
- [x] A-5: ch03 `.rela.text 0x178` → `0x180`
- [x] A-6: ch03 readelf -l a.out を実機値に
- [x] A-7: ch08 助詞ミス
- [x] A-8: ch08 擬似コード API 名を `Elf::parse` に
- [x] B-1: ch05 ディレクトリ図 `mod.rs` → `linker.rs`
- [x] B-2: ch05 `ELF_IDENT_HEADER_SIZE` 命名整理
- [x] B-3: ch06 `Binding` enum に Weak 注記
- [x] B-4: ch09 `is_stronger_than` (Local, Local) 注記
- [x] B-5: ch10 `shndx` 判定ハックの注記または修正
- [x] B-6: ch10/11 `apply_relocations` 引数型統一
- [x] C-1: ch08 `.text` オフセット 0x100 の内訳
- [x] C-2: ch10 `text_offsets` の役割解説
- [x] C-3: ch11 `0x9F00001F` マスクの導出
- [x] C-4: ch12 `.symtab` 規約解説
- [x] C-5: ch09 Weak 注記
- [x] C-6: ch13 参考資料（書籍紹介）
