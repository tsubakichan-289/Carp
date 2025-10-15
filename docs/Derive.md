# `derive` の使い方

`derive` は、データ型のメンバーに基づいて自動的にインタフェース実装を生成する仕組みです。また、自分で `derive` の振る舞いを定義する「`deriver`」を作成することもできます。

自作の型に対してインタフェースを `derive` する方法を知りたい場合は[第1節](#i-using-derive)を、インタフェース用の `deriver` を提供したい場合は[第2節](#ii-writing-derivers)をお読みください。

## I: `derive` を利用する

通常、`derive` の利用は型名と実装したいインタフェースを渡して呼び出すだけです。

```clojure
(deftype Point [
  x Int
  y Int
])
(derive Point zero)
(derive Point =)

; 別名で関数を生成したい場合は第三引数を指定
; 名前衝突を避けたいときなどに便利
(derive Point str my-str)
```

上記のコードは、`Point` 型のメンバーに基づいて `zero` と `=` の実装を自動生成します。これが成立するためには、型が**具体型であること**（型変数を含まない）と、**メンバー自身が対象のインタフェースを実装していること**が必要です。`zero` の定義はメンバーの `zero` を組み合わせることで成り立ち、`=` も同様にメンバーの比較に依存するためです。

Carp が標準で提供する自動導出は `=`, `zero`, `str` の 3 つだけです。ただし依存するライブラリが独自の `deriver` を提供している場合もあります。利用可能な `deriver` は `(derivables)` で確認でき、特定のインタフェースが導出可能かは `(derivable? <interface>)` で調べられます。インタフェース名は引用符付きで渡してください。

上記の前提のいずれかを満たさない場合は、自分で関数を実装する必要があり、`derive` は使えません。

ある型に対して、メンバー全体に同じ処理を適用してその型を返す「更新系」のインタフェースを導出したいケースもあります。`Point` における `inc` がその代表例です。

通常は自作の `deriver` を書く必要があります（詳しくは [第2節](#ii-writing-derivers) を参照してください）が、Carp には `make-update-deriver` という特別な動的関数が用意されています。値を更新して返す単項インタフェースを渡すと、包含する型に対する定義を導き出してくれます。`Point` に対して利用する例は以下の通りです。

```clojure
(make-update-deriver 'inc) ; notice the quote
(derive Point inc)

(inc (Point.zero)) ; => (Point 1 1)
```

これは便利な場合がありますが、上記のような特殊なケースに限定されます。`update-<member>` 系の関数に渡せるような関数に対してのみ利用できます。

## II: `deriver` を作る

標準で提供されていないインタフェースに対して独自の導出ロジックを用意したいこともあります。その場合は `make-deriver` を使って自前の `deriver` を定義できます。

動的関数 `make-deriver` は 3 つの引数を取ります。インタフェース名（引用符付き）、呼び出し時に渡される引数名のリスト、そして型を受け取ってその型向けの実装を生成する関数です。

イメージしづらいので、`zero` 用の `deriver` を例に見てみましょう。

```clojure
(make-deriver 'zero []
  (fn [t]
    (cons 'init
      (map (fn [_] '(zero)) (members t)))))
```

`make-deriver` は関数定義のように読むと理解しやすいです。ここではインタフェース名が `zero` で引数はありません。型が渡されたときに、そのメンバーそれぞれに `zero` を呼び出して `init` で包めば `zero` の定義になる、ということを表現しています。したがって `Point` 向けの `zero` は次のようになります。

```clojure
(init (zero) (zero))
```

`derive` 自身が周辺のボイラープレートを生成するため、`(derive Point zero)` の呼び出し全体は次のように展開されます。

```clojure
(defmodule Point
  (defn zero []
    (init (zero) (zero)))
  (implements Point.zero zero)
)
```

つまり `deriver` が知っておくべきことは、型が与えられたときにどのような関数本体を生成するかだけです。引数名も制御できるため、必要に応じて生成するコードの中で活用できます。
