# `drop` インタフェース

Carp の任意の型が実装できる「特殊な」インタフェースのひとつに `drop` があります。シグネチャは `(Fn [&a] ())` で、型への参照を受け取り、その値が破棄される直前に呼び出されます。ファイルのクローズのように解放時に特別な処理が必要な型で利用します。

```clojure
(deftype A [])

(defmodule A
  (sig drop (Fn [(Ref A)] ()))
  (defn drop [a]
    (IO.println "Hi from drop!"))
)

(defn main []
  (let [a (A)]
    ()))
```

上記の例では、`let` のスコープを抜けるタイミングで `A.drop` が実行され、「Hi from drop」が出力されます。
