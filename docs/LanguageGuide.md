## 言語リファレンス

### はじめに

Carp の構文は Clojure に似ていますが、実行時のセマンティクスは ML や Rust に近いものです。型は基本的に推論されますが、可読性のために `the` キーワードで注釈を付けることもできます（後述）。

メモリ管理は静的解析で行われ、値は生成された関数に所有権があります。値を戻り値として返したり、別の関数へ渡した場合は元の関数が所有権を放棄し、その後に使用するとコンパイルエラーになります。別の関数に一時的に値を貸し出したい場合（たとえば表示したいとき）は `ref` 特殊フォーム（またはリーダーマクロ `&`）を使って参照を生成します。

メモリ管理の詳細は [Memory.md](Memory.md) を参照してください。

### コメント
```clojure
;; セミコロンから行末までがコメントです。
```

### データリテラル
```clojure
100     ;; Int
1500l   ;; Long
3.14f   ;; Float
10.0    ;; Double
1b      ;; Byte
true    ;; Bool
"hello" ;; &String
#"hello" ;; &Pattern
\e      ;; Char
[1 2 3] ;; (Array Int)
{1 1.0 2 2.0} ;; (Map Int Double)
```

### 型リテラル
```clojure
t ;; 型変数は小文字で始まります
(f t) ;; 型コンストラクタの変数。`(Maybe Int)` にはマッチするが `Int` にはマッチしない
Int
Long
Float
Double
Byte
Bool
String
Pattern
Char
(Array t)
(Map <key-type> <value-type>)
(Fn [<arg-type1> <arg-type2> ...] <return-type>) ;; 関数型
```

### 動的専用のデータリテラル
現在、以下のデータ型は非コンパイルコード（REPL など）でのみ操作可能です。

```clojure
(1 2 3) ; list
foo ; symbol
```

### 定義の仕方
```clojure
(defn function-name [<arg1> <arg2> ...] <body>) ;; 関数定義（コンパイルされ、REPL からは呼び出せない）
(definterface interface-name (Fn [<t1> <t2>] <return>)) ;; 複数の実装を持てる汎用関数を定義
(def variable-name value) ;; グローバル変数を定義（現状はプリミティブな定数を想定）
(defmacro <name> [<arg1> <arg2> ...] <macro-body>) ;; マクロ定義。引数は展開時に評価されない
(defdynamic <name> <value>) ;; REPL やコンパイル時のみ利用できる変数
(defndynamic <name> [<arg1> <arg2> ...] <function-body>) ;; REPL やコンパイル時のみ利用できる関数
(defmodule <name> <definition1> <definition2> ...) ;; プログラムをモジュールに分割する主要な手段
```

### `cond` による条件分岐
`cond` は条件が真なら対応するコードブロックを実行し、偽なら別のブロックを実行します。

```clojure
(doc cond "this is the documentation for cond")
```

使用例:

```clojure
(cond
          (<condition_1>) (<code_1>) ;; condition_1 が真なら code_1 を実行
          (<condition_2>) (<code_2>) ;; condition_2 が真なら code_2 を実行
          (<code_3>) ;; 上記がどちらも偽なら code_3 を実行
```

10 より小さいか大きいかでメッセージを切り替える例:

```clojure
(cond
  (< 10 1) (println "Don't print!")
  (> 10 1) (println msg)
  (println "Don't print!"))
```

### 特殊フォーム
以下のフォームは Carp のソースコードで利用でき、型チェックや静的解析の後に C へ変換されます。最初の 3 つは動的関数でも使用可能です。

```clojure
(fn [<arg1> <arg2> ...] <body>) ;; ラムダ（クロージャ）を生成
(let [<var1> <expr1> <var2> <expr2> ...] <body>) ;; ローカル変数を束縛
(do <expr1> <expr2> ... <return-expression>) ;; 副作用のある処理を順に実行し、最後の値を返す
(if <expression> <true-branch> <false-branch>) ;; 条件分岐
(while <expression> <body>) ;; 式が偽になるまでループ
(use <module>) ;; モジュール内のシンボルをスコープへ取り込む
(with <module> <expr1> <expr2> ...) ;; ローカルな `use`。以降の式で <module> のシンボルを参照
(match <expression> <case1> <expr1> <case2> <expr2> ...) ;; 合併型コンストラクタに対してパターンマッチ
(match-ref <expression> <case1> <expr1> <case2> <expr2> ...) ;; 参照型の値を所有権を奪わずにパターンマッチ
(ref <expression>) ;; 所有権を持つ値を借用
(set! <variable> <expression>) ;; 変数を破壊的に更新
(the <type> <expression>) ;; 式の型を明示的に指定
```

例えば、`the` フォームを使って Int のみを受け付ける恒等関数を定義できます。

```clojure
(defn f [x]
  (the Int x))
```

### リーダーマクロ
```clojure
&x ;; (ref x) と同じ
@x ;; (copy x) と同じ
```

### 名前付きホール
静的型付き言語では、どんな値を置けばよいか分からなくなることがあります。そのようなときは「ホール」を置いておき、`:r`（リロード）するとコンパイラが必要な型を教えてくれます。

```clojure
(String.append ?w00t "!") ;; '?w00t' の型が &String だと分かる
```

### 動的コード評価時の特殊フォーム
```clojure
(quote <expression>) ;; これ以上評価しない
(and) (or) (not) ;; 論理演算子
```

### 動的関数
これらは REPL もしくはマクロ展開時にのみ使用できます。よく使われる一部を紹介します。

```clojure
(car <collection>) ;; リスト／配列の先頭要素を返す
(cdr <collection>) ;; 先頭要素以外を返す
(cons <expr> <list>) ;; <expr> の値を <list> の先頭へ追加
(cons-last <expr> <list>) ;; <expr> の値を <list> の末尾へ追加
(list <expr1> <expr2> ...) ;; 評価した式列からリストを生成
(array <expr1> <expr2> ...) ;; 評価した式列から配列を生成
```

`Dynamic` モジュールで利用できる関数の一覧は REPL で `(info Dynamic)` を実行してください。

### 構造体（Structs）
Carp で定義した構造体はすべて `init` メソッドを持ち、新しいインスタンスを生成できます。フィールドは定義順に渡す必要があります。

```clojure
(deftype Vector2 [x Int, y Int])

(let [my-pos (Vector2.init 10 20)]
  ...)

;; 各メンバーには自動的にレンズが生成されます
;; Vector2.x (Fn [(Ref Vector2)] (Ref Int))
(Vector2.x &my-pos) ;; => 10
;; Vector2.set-x (Fn [Vector2 Int] Vector2)
(Vector2.set-x my-pos 30) ;; => (Vector2 30 20)
;; Vector2.set-x! (Fn [(Ref Vector2), Int] ())
(Vector2.set-x! &my-pos 30) ;; => その場で更新し () を返す
;; 関数への参照を取るレンズ
;; Vector2.update-x (Fn [Vector2, (Ref (Fn [Int] Int))] Vector2)
(Vector2.update-x my-pos inc) ;; => (Vector2 11 20)
;; ラムダも渡せます
(Vector2.update-x my-pos &(fn [n] (* n 3))) ;; => (Vector2 30 20)
```

### 合併型（Sumtype）
`sumtype` を定義する方法は 2 通りあります。

**列挙型:**
```clojure
(deftype MyEnum
  Kind1
  Kind2
  Kind3)
```

**データ型:**
```clojure
(deftype (Either a b)
  (Left [a])
  (Right [b]))
```

バリアントは関数呼び出しのような構文で生成します。
```clojure
(MyEnum.Kind1)
(Either.Left 10)
(Either.Right 11)

;; `use` しておけばモジュール名を省略可能
(use Either)
(Left 10)
(Right 11)

(use MyEnum)
(Kind1)
(Kind2)
(Kind3)
```

値を安全に取り出すにはパターンマッチを利用します。
```clojure
(defn get [either]
  (match either
    (Either.Left a) a
    (Either.Right b) b))

(with MyEnum
  ;; 「その他」のケースをまとめることもできます
  (match myenum
    (Kind1) (logic1)
    _ (logic-other)))
```

`match` は値（所有権を持つもの）に対して動作するため、マッチ対象の所有権を取得します。参照に対してマッチしたい場合は `match-ref` を使います。

```clojure
(match-ref &might-be-a-string
  (Just s) (IO.println s)
  Nothing (IO.println "Got nothing"))
```

このコードは `might-be-a-string` の所有権を奪いません。また最初のケースで得られる `s` は参照です。この状況では `Maybe` を値として分解するのは安全ではないためです。

**注意:** 合併型は最大 128 個までのバリアント（コンストラクタ）しか持てません。バイト数の制限を想像した方はその通りです。制約ではありますが、現在のところ問題になるケースは報告されていません。

### モジュールと名前解決
関数や変数はモジュールに格納でき、モジュールはネスト可能です。モジュール内のシンボルを使うには `Float.cos` のようにモジュール名で修飾します。

`use` を使うと修飾なしで参照できます。

```clojure
(use Float)

(defn f []
  (cos 3.2f))
```

複数のモジュールを `use` し、同名のシンボルが存在すると型推論器が文脈から解釈を試みます。判断できない場合はエラーになります。たとえば `String` と `Array` にはどちらも `length` 関数が存在しますが、以下のコードでは配列版が必要だと分かるため問題ありません。

```clojure
(use String)
(use Array)

(defn f []
  (length [1 2 3 4 5]))
```

一方、次のコードでは意図が判別できずエラーになります。
```clojure
(use String)
(use Array)

(defn f [x]
  (length x))
```

型を明示すれば解決できます。
```clojure
(use String)
(use Array)

(defn f [x]
  (String.length x))
```

グローバルスコープで `use` すると、そのモジュールの宣言がグローバルスコープへ取り込まれます。モジュール内部で `use` した場合は、そのモジュールのスコープ内で修飾なしに参照できます。

```clojure
(use String)

;; グローバルでは String だけを use しているので修飾なしで呼べる
(defn f [x]
  (length x))

(defmodule Foo
  (use Array)
  ;; グローバルでは String を use、Foo では Array を use しているため
  ;; `length` を呼ぶ際は曖昧さを避けるため修飾が必要
  (defn g [xs]
    (Array.length xs)))
```

一定範囲だけ宣言を取り込みたい場合は `with` フォームが便利です。

```clojure
(defmodule Foo
  ;; ここではまだ修飾が必要
  (defn f [x]
    (String.length x))

  ;; with のスコープ内では修飾なしで参照できる
  (with String
    (defn g [x]
      (length x))))
```

バインディングをモジュール内部に隠しておきたい場合は `private` と `hidden` を使います。

```clojure
(defmodule Say
  ;; モジュール外から `hell` を参照できなくする
  (private hell)
  ;; バインディング一覧で `hell` が表示されないようにする
  (hidden hell)
  (defn hell [] @"hell")

  ;; `def` や `defn` に対しても利用可能
  (private o)
  (hidden o)
  (def o @"o")

  ;; モジュール内からは参照できる
  (defn hello [] (String.concat &[(hell) @&o])))


(Say.hello) ;; `hello` は公開されているので呼べる

(Say.hell) ;; `hell` は private のためコンパイルエラー
```

`defn-` や `def-` を使うと、バインディング定義と同時に `private`／`hidden` を付与できます。以下は先ほどのコードと同等です。

```clojure
(defmodule Say
 (defn- hell [] @"hell")
 (def- o @"o")
 (defn hello [] (String.concat &[(hell) @&o])))
```

### インタフェース

インタフェースは汎用関数のシグネチャを定め、複数の具象関数がその実装を提供できます。`definterface` に名前と型シグネチャを渡して定義します。

```clojure
(definterface speak (Fn [a] String))
```

実装を宣言するには `implements` を使用します。次の例では `Dog.bark` と `Cat.meow` を `speak` の実装として登録しています。

```clojure
(definterface speak (Fn [a] String))

(defmodule Dog
  (defn bark [aggressive?]
    (if aggressive? @"WOOF!" @"woof!"))
  (implements speak Dog.bark))

(defmodule Cat
  (defn meow [times] (String.repeat times "meow!"))
  (implements speak Cat.meow))
```

インタフェースのシグネチャと一致する関数のみが実装として登録できます。以下のように引数や戻り値が合わない場合はエラーになります。

```clojure
(defmodule Number
  (defn holler [] "WOO!")
  (implements speak Number.holler))
=> [INTERFACE ERROR] Number.holler : (Fn [] (Ref String a)) doesn't match the interface signature (Fn [a] String)
```

インタフェース名で呼び出すと、現在のコンテキストと各実装の型シグネチャをもとに適切な実装が選ばれます。

```clojure
(speak 2) ;; Int -> String。Cat.meow
=> "meow!meow!"
(speak false) ;; Bool -> String。Dog.bark
=> "woof!"
```

同じシグネチャの実装が複数存在し曖昧になる場合はエラーになります。

```clojure
(defmodule Pikachu
  (defn pika [times] (String.repeat times "pika!"))
  (implements speak Pikachu.pika))

(speak 2)
=> There are several exact matches for the interface `speak` of type `(Fn [Int] String)` ...
```

この場合は、必要な実装を直接呼び出して曖昧さを解消します。同じシグネチャの実装を複数提供するのはあまり有用ではありません。

### C との相互運用
```clojure
(system-include "math.h") ;; #include <math.h> に展開
(relative-include "math.h") ;; #include "$carp_file_dir/math.h" に展開（$carp_file_dir は .carp のディレクトリ絶対パス）

(register blah (Fn [Int Int] String)) ;; Int を 2 つ受け取り String を返す C 関数 `blah` を登録
(register pi Double) ;; Double 型のグローバル変数 `pi` を登録

(register blah (Fn [Int Int] String) "exit") ;; C コード上の名前を `exit` にする

(register-type Apple) ;; Opaque な C 型を登録
(register-type Banana [price Double, size Int]) ;; 外部の C 構造体を登録（getter/setter/updater を自動生成）
```

C の型名は小文字であることが多く（`size_t` など）、そのまま登録すると Carp では多相型と解釈されるため問題が生じます。その場合は `register-type` の第 2 引数に C 側の名前を渡します。

```clojure
(register-type SizeT "size_t")
```

これにより Carp では `SizeT` として扱い、出力される C コードでは `size_t` になります。

詳しくは [C との相互運用ガイド](./CInterop.md) を参照してください。

### パターン

パターンは正規表現に似ていますが同じではありません。[Lua](http://lua-users.org/wiki/PatternsTutorial) に由来し、文字列から何かを検索・抽出したいときに便利です。正規表現ほど表現力はありませんが、そのぶん高速かつ予測しやすいのが特徴です。

API の概要は以下のとおりです。

```clojure
; リテラルで初期化するか、文字列から生成
#"[a-z]"
(Pattern.init "[a-z]")

; 文字列に戻すことも可能
(str #"[a-z]")
(prn #"[a-z]")

; 文字列内の位置を検索
(Pattern.find #"[a-z]" "1234a") ; => 4
(Pattern.find #"[a-z]" "1234")  ; => -1

; 複数ヒットも
(Pattern.find-all #"[a-z]" "1234a b") ; => [4 6]

; matches? は完全一致を確認
(Pattern.matches? #"(\d+) (\d+)" "  12 13") ; => true

; match-groups は最初のマッチのグループを返す
(Pattern.match-groups #"(\d+) (\d+)" "  12 13") ; => ["12" "13"]

; match-str は最初のマッチ全体を返す
(Pattern.match-str #"(\d+) (\d+)" "  12 13") ; => "12 13"

; global-match は全マッチのグループを取得
(Pattern.global-match #"(\d+) (\d+)" "  12 13 14 15") ; => [["12" "13"] ["14" "15"]]

; substitute は文字列内のパターンを n 回置換
(Pattern.substitute #"sub-me" "sub-me sub-me sub-me" "replaced" 1) ; => "replaced sub-me sub-me"

; すべて置換するなら -1
(Pattern.substitute #"sub-me" "sub-me sub-me sub-me" "replaced" -1) ; => "replaced replaced replaced"
```

#### パターンの制限

前述のとおり、パターンは正規表現ほど表現力がありません。最も大きな違いはバックトラックをしないことです。そのため選択（オルタネーション）を表現できず、非貪欲な左側マッチも縮められません。後者は直感的ではないかもしれないので例を見てみましょう。

```
(Pattern.match-all-groups #"1.-2" "1 1 2") ; => [[@"1 1 2"]]
```

より短いマッチである `"1 2"` も理論上は正しいですが、左側に戻ってマッチ範囲を縮める必要があるため採用されません。このように `-` は正規表現の `*?` に似ていますが同じではありません。多くの場合、より明示的なパターンを用意すれば解決できます（上記なら `#"1\s-2"` など）。
