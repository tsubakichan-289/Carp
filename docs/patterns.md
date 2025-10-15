# Carp のパターン集

このドキュメントでは Carp でよく登場するプログラミングパターンを紹介します。

## Ref-let

スコープ内の非参照引数に対して、所有権を奪うタイプの関数を 2 回適用したいことがあります。次の関数を考えてみましょう。

```clojure
(defn needs-ref-let [x]
  (Pair.init (Maybe.or-zero x) (Maybe.or-zero x)))
```

`Maybe.or-zero` は参照でない管理対象引数 `x` の所有権を取得するため、上記のコードはコンパイルエラーになります。これは `x` に紐づくメモリの破壊的変更を防ぎ、多重スレッド環境で `Maybe.or-zero` が同時に `x` へアクセスするような状況でも安全性を保つためです。

それでも単一の `x` に対して 2 回 `Maybe.or-zero` を実行したい場合があります。そのような場合は、`x` の参照を束縛して使い回すのが便利です。

```clojure
(defn needs-ref-let [x]
  (let [x* &x]
    (Pair.init (Maybe.or-zero @x*) (Maybe.or-zero @x*))))
```

ここで `x*` は `x` の参照であり、関数本体の中で自由にコピーできます。実際には 2 回目のコピーは不要で、1 度コピーしたあとは元の `x` をそのまま使えます。

```clojure
(defn needs-ref-let [x]
  (let [x* &x]
    (Pair.init (Maybe.or-zero @x*) (Maybe.or-zero x))))
```

---
1: Carp では `Maybe` のような合併型は借用チェッカの管理対象です。つまり、対応するメモリの所有権や解放タイミングを Carp が追跡します。すべての型が管理対象というわけではありません。例えば `Int` は管理対象ではないため、上記のような問題は発生しません。詳しくは [Memory.md](Memory.md) を参照してください。
