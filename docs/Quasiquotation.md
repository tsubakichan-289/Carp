# クォジクォート（準クォート）

準クォートは、リストの一部を評価しつつ残りをクォートしたまま扱う仕組みで、動的な文脈でのみ利用できます。

準クォートを使うと、評価済み（アンコートされた）要素と未評価（クォートされた）要素を同じリスト内に混在させることができます。

```clojure
(defdynamic x 2)

(quasiquote (+ (unquote x) 1)) ; => (+ 2 1)

; リテラル表記も利用できます。quasiquote は `、unquote は % に展開されます。
`(+ %x 1) ; => (+ 2 1)
```

`unquote` は `quasiquote` の内部でのみ意味を持ち、外側で使うとマクロ展開時にエラーになります。

準クォートは主にリストを対象とするため、別のリストの要素をフラットに差し込みたい（スプライスしたい）こともあります。Carp ではその用途向けに `unquote-splicing` を提供しています。

```clojure
(defdynamic x '(1 2))

(quasiquote (+ (unquote-splicing x))) ; => (+ 1 2)

; リテラル表記では unquote-splicing は %@
`(+ %@x) ; => (+ 1 2)
```

上記の例では変数のみを使っていますが、`unquote` 系のフォームの内部には任意の式を記述できます。

```clojure
(quasiquote (+ (unquote-splicing (map inc [1 2])))) ; => (+ 2 3)
; もしくは
`(+ %@(map inc [1 2])) ; => (+ 2 3)
```
