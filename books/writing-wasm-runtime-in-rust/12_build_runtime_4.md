---
title: "Runtimeの実装 ~ \"Hello, World\"出力まで ~"
---

本章では`Wasm Runtime`が持つメモリにある`Hello, World`を出力する機能を実装する。
標準出力に文字列を出力するため`WASI`関数である`fd_write`も実装する。

最終的には次のWATを実行できるようになる。

```WAT:src/fixtures/hello_world.wat
(module
  (import "wasi_snapshot_preview1" "fd_write"
    (func $fd_write (param i32 i32 i32 i32) (result i32))
  )
  (memory 1)
  (data (i32.const 0) "Hello, World!\n")

  (func $hello_world (result i32)
    (local $iovec i32)

    (i32.store (i32.const 16) (i32.const 0))
    (i32.store (i32.const 20) (i32.const 14))

    (local.set $iovec (i32.const 16))

    (call $fd_write
      (i32.const 1)
      (local.get $iovec)
      (i32.const 1)
      (i32.const 24)
    )
  )
  (export "_start" (func $hello_world))
)
```

## `Memory Section`のデコード実装


## `Data Section`のデコード実装


## 必要な命令の実装

今回のWATを実行するのに以下の命令を処理できるようにする必要があるので、それを実装していく。

- i32.const
- local.set

## `WASI`の`fd_write`の実装

## まとめ
これで`"Hello, World"`が出力できる小さな`Wasm Runtime`の完成だ。
