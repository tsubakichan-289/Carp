# C との相互運用

この文書は [Language Guide](./LanguageGuide.md#c-interop) の内容を補足するものです。

## 目次

- [Carp が識別子を生成する仕組み](#how-carp-generates-identifiers)
- [管理対象型](#managed-types)
  - [String](#string)
  - [Array](#array)
- [Carp に C コードを埋め込む](#embedding-c-code-in-carp)
  - [`deftemplate`](#deftemplate)
    - [`基本例`](#basic-example)
    - [`ジェネリクス`](#generics)
  - [`emit-c`](#unsafe-emit-c)
  - [`preproc`](#unsafe-preproc)
  - [型の登録](#register-types)
- [コールバック](#callbacks)
- [Headerparse](#headerparse)

## Carp が識別子を生成する仕組み

関数や `def` を定義する際、C 側でどのような識別子が生成されるのかを知っておくと便利です。以下にいくつかの例を示します。

```clojure
(def a-def 100)
; => a_MINUS_def

(defn hello [] (println* "Hello"))
; => hello

(sig true? (Fn [Bool] Bool))
(defn true? [b] b)
; true_QMARK_

(defmodule Reverse
  (defn hello [] (println* "Goodbye"))
  ; => Reverse_hello
  (defmodule ReReverse
    (defn hello [] (println* "Hello"))))
    ; => Reverse_ReReverse_hello

; Generic signature
(sig print-first-and-add (Fn [(Ref (Array a)) b b] b))
(defn print-first-and-add [arr x y]
 (do
   (println* (Array.unsafe-first arr))
   (+ x y)))
; Generates no code until it is called

(print-first-and-add &[1] 1 1)
; => print_MINUS_first_MINUS_and_MINUS_add__int_int
(print-first-and-add &[@"hello"] 2l 40l)
; => print_MINUS_first_MINUS_and_MINUS_add__String_Long
```

例を見ると、Carp が C コード生成前に識別子をどのように変換するかがわかります。ポイントをまとめると次のようになります。

- C にとって不正な文字は文字列に展開されます（`- => _MINUS_`、`? => _QMARK_` など）。
- モジュール内で定義された場合は、モジュール名が接頭辞として付与されます。
- 関数の引数がジェネリックな場合、利用時に型名が接尾辞として付与されます。実際に使用されるまで識別子は生成されません。
- ジェネリックにしたくない場合は、`true?` の例のように非ジェネリックな型シグネチャを付けることで、一般化しない識別子を生成させられます。

このプロセスは *マングリング* と呼ばれ、Carp では正しいが C では不正な識別子によって無効な C コードが生成されてしまう状況を避けるために必要です。

### Overriding Carp's default C identifier names

既存の C ライブラリに対するバインディングを Carp で作成する際、C 側の識別子を完全に一致させるのは面倒です。マングリングの影響で、Carp のバインディングをモジュールでラップすると接頭辞が付いてしまい、C 側の識別子と一致しなくなる可能性があります。こうした手間を減らすため、`register` および `register-type` には C 側で使用する識別子を明示するためのオプション引数があります。

```clojure
(defmodule CURL
  (register-type HttpPost "curl_httppost")
  (register form-free (Fn [(Ref HttpPost)] ()) "curl_formfree"))
```

この仕組みによって、Carp 側では自由な構造（例では cURL バインディングを `CURL` モジュールにまとめています）を保ちながら、生成される C 側の識別子を既存ライブラリに合わせられます。例えば `form-free` という Carp 側の識別子は通常 `form_MINUS_free` にマングリングされますが、オーバーライドを指定すれば `curl_formfree` として出力されます。

同様に、Carp のみで定義されたコードに対しても C 側の識別子を上書きできます。既存の C プログラム内の安全性が重要な部分を Carp に移植し、生成された安全な C コードを元のプログラムから呼びたい場合など、ネストされたモジュールやカスタム型、特殊文字を多用しているとマングリング結果が煩雑になりがちです。

そのようなときは `c-name` メタフィールドを利用して、特定の定義に対する C 側の識別子を明示的に指定できます。これにより生成される C コードが読みやすくなり、他言語からも呼び出しやすくなります。例えば次のようにします。

```clojure
(defn foo-bar [] 2)
(c-name foo-bar "foo_bar")
```

これにより、通常であれば `foo_MINUS_bar` となるところが `foo_bar` という識別子で C コードが生成されます。

## Managed types

Carp における `String` や `Array` といった型はヒープ上に確保され、スコープから外れたタイミングでコンパイラが自動的に解放する *管理対象* の型です。ここでは、それら管理対象型を C 側とやり取りする方法を見ていきます。

### String

管理対象の `String` を `char*` を期待する C 関数に渡すには、`String.cstr` を使って `(Ref String)` を `(Ptr CChar)` に変換します。


```clojure
(register puts (Fn [(Ptr CChar)] ()))

(let [a-str @"A string."]
  (puts (String.cstr &a-str)))
(puts (String.cstr "Hello"))
```

エンドユーザに C の型を意識させたくない場合は、次のようにラップできます。

```clojure
(defmodule MyMod
  (hidden puts-c)
  (private puts-c)
  (register puts-c (Fn [(Ptr CChar)] ()) "puts")
  (defn puts [str-ref] (puts-c (String.cstr str-ref))))

(let [a-str @"A string."]
  (MyMod.puts &a-str))
(MyMod.puts "Hello")
```

---

`char*` を受け取り管理対象の `String` に変換したい場合は `String.from-cstr` を使用します。この関数は C 文字列の内容を確保・コピーします。

```c
// static-str.h
char* returns_a_static_str() {
  return "Hello";
}
```

```clojure
(relative-include "static-str.h")

(register returns-a-static-str (Fn [] (Ptr CChar)) "returns_a_static_str")

(let [a-str (String.from-cstr (returns-a-static-str))]
  (println* (String.concat &[a-str @" " @"Carp"])))
```

---

呼び出す C 関数側でヒープに文字列を確保しているケースもあります。その場合、戻り値を管理対象の `String` として宣言できます。ただし、実際にヒープ確保されていること、そして Carp が利用しているアロケータと同じものであることを確認する必要があります。条件が満たされなければ安全ではありません。

```c
char* returns_a_heap_string() {
  char *hello = "Hello from the heap";
  char *str = malloc((strlen(hello)+1));
  strcpy(str, hello);
  return str;
}
```

```clojure
(relative-include "heap-string.h")

(register returns-a-heap-str (Fn [] String) "returns_a_heap_string")

(let [a-str (returns-a-heap-str)]
  (println* a-str))
```

C コード側を自分で記述できるなら、`CARP_MALLOC` マクロを利用することで Carp コンパイラと同じアロケータを使用できます。

```c
char* returns_a_heap_string() {
  char *hello = "Hello from the heap";
  char *str = CARP_MALLOC((strlen(hello)+1));
  strcpy(str, hello);
  return str;
}
```

### Array

引数として C の配列を受け取る関数に渡す場合は `Array.unsafe-raw` を使えます。

```c
int sum(int *arr, int len) {
  int acc = 0;
  for (int i = 0; i < len; i++) {
    acc += arr[i];
  }
  return acc;
}
```

```clojure
(relative-include "sum.h")

(register sum-c (Fn [(Ptr Int) Int] Int) "sum")

(let [ints [1 2 3]]
  (println* (sum-c (Array.unsafe-raw &ints) (Array.length &ints))))
```

ここでも、素の C 関数を Carp らしいインタフェースでラップできます。

```clojure
(relative-include "sum.h")

(defmodule MyMod
  (hidden sum-c)
  (private sum-c)
  (register sum-c (Fn [(Ptr Int) Int] Int) "sum")

  (sig sum (Fn [(Ref (Array Int))] Int))
  (defn sum [ints] (sum-c (Array.unsafe-raw ints) (Array.length ints))))

(MyMod.sum &[1 2 3])
```

---

消費側の関数がデータの所有権を引き受ける場合は `Array.raw` を使用します。その場合、`free` を呼び出し、内部にある管理対象型も適切に解放する責任は渡した側ではなく受け取った側にあります。

```c
// printall.h
void println_all(char **arr, int len) {
  for (int i = 0; i < len; i++) {
    printf("%s\n", arr[i]);
    CARP_FREE(arr[i]);
  }
  CARP_FREE(arr);
}
```

```clojure
(relative-include "printall.h")

(register println-all (Fn [(Ptr String) Int] ()) "println_all")

(let [lines [@"One" @"Two" @"Three"]
      len (Array.length &lines)]
  (println-all (Array.raw lines) len))
```

## Carp に C コードを埋め込む

C ライブラリにバインディングするとき、ライブラリの関数をカスタム C コードでラップすると便利な場合があります。最も単純な方法は、ヘッダファイルに C コードを書き、Carp 側から `include` して `register` することです。

```c
// print.h
// String is a carp core alias for char*
void print_that_takes_ownership(String str) {
  printf("%s", str);
  CARP_FREE(str);
}
```

```clojure
(relative-include "print.h")

(register print (Fn [String] ()) "print_that_takes_ownership")

(print @"Print this!")
```

とはいえ、C コードを Carp コードの近くに置いておきたいこともあります。そこで `deftemplate` の出番です。

### `deftemplate`

#### 基本例

先ほどの例は次のようにも記述できます。

```clojure
(deftemplate print (Fn [String] ())
                   "void $NAME(String str)"
                   "$DECL {
                     printf(\"%s\", str);
                     CARP_FREE(str);
                   }")

(print @"Print this!")
```

ここで何が起こっているかを分解してみましょう。

- `deftemplate` の **第1引数** は Carp 側で参照する関数名です。
- **第2引数** は型シグネチャで、先ほど `register` で指定したものと同じです。
- **第3引数** は関数宣言で、生成される C ファイルの先頭に挿入されます。
- **最後の引数** が関数本体に相当します。

注目すべき点がもう 2 つあります。

- `$NAME` は与えた関数名とモジュール名から導出されるため、他のモジュールで定義した `print` と名前が衝突する心配はありません。
- `$DECL` は第3引数で渡した宣言に置き換えられます。

このように `deftemplate` を使うと Carp と C のコードを近くに保ち、全体的な記述量も減らせます。ただし真価は別のところにあります。

#### Generics

2 つの数値を加算する関数を書きたいとします。型ごとに別々のバージョンを書くのは骨が折れますが、`deftemplate` を使えば簡単に一般化できます。

```clojure
(deftemplate add (Fn [a a] a)
                 "$a $NAME($a x, $a y)"
                 "$DECL {
                   return x + y;
                 }")

(add 1 2)
(add 20l 22l)
(add 2.0f 5.0f)

; Can't do that as they're different types
; (add 2.0f 22l)
```

Carp では型シグネチャにジェネリック型（例では `a`）を使えます。C コード内では `$` に続けてジェネリック名を書くことで同じ型を参照できます。テンプレートが異なる型で利用されるたび、Carp は専用の関数を生成します。

ただし注意が必要です。この方法では Carp コンパイラが提供していた型安全性を失います。C コンパイラが間違いを検出してくれることを願うしかありません。

``` clojure
(deftemplate add (Fn [a a] a)
                 "$a $NAME($a x, $a y)"
                 "$DECL {
                   return x + y;
                 }")

(add @"A string" @" another string")
```

幸いにもこの例では Clang がエラーを出してくれますが、常に頼れるとは限りません。

```
out/main.c:9153:29: error: invalid operands to binary expression ('String' (aka 'char *') and 'String')
                   return x + y;
                          ~ ^ ~
1 error generated.
```

### `Unsafe.emit-c`

`deftemplate` は柔軟で多くの場合に対応できますが、万能ではありません。例えば C11 の `static_assert` のように引数として文字列リテラルを要求するマクロがあります。`deftemplate` ではこうしたリテラルを直接挿入できないため、`Unsafe.emit-c` を使って文字列リテラルをそのまま出力させます。`static_assert` を `static-assert` として `register` しているとすると、以下のように書くことで生成 C コードに文字列リテラルを埋め込めます。

```
(register static-assert (Fn [a C] ()))

(static-assert 0 (Unsafe.emit-c "\"foo\""))
```

これにより次のような C コードが出力されます。

```
static_assert(0, "foo")
```

`emit-c` の戻り値は `C` 型です。`C` は Carp においてリテラルな C コードを表現するための特別な型です。

### `Unsafe.preproc`

Carp のコンパイラは、関数の依存関係が解決されるよう適切な順序で C コードを出力します。しかし、出力の前に任意の C コードを挿入したいこともあります。プリプロセッサディレクティブを追加したい場合などが典型です。`Unsafe.preproc` はこの用途のために用意されています。`preproc` で渡したコードはファイルの `include` の直後、その他の出力よりも前に挿入されます。

`preproc` の引数は `C` 型なので、`Unsafe.emit-c` と組み合わせて使う必要があります。渡した C コードの内容は一切検査されないため、扱いには注意してください。

`preproc` で C のシンボルを定義した場合でも、Carp から参照するには `register` を呼ぶ必要があります。以下の例では C のマクロと関数を `preproc` で定義し、その後 `register` を使って Carp から参照しています。

```
(Unsafe.preproc (Unsafe.emit-c "#define FOO 0"))
(Unsafe.preproc (Unsafe.emit-c "void foo() { printf(\"%d\\n\", 1); }"))

(register FOO Int)
(register foo (Fn [] ()))

(defn main []
  (do (foo)
      (IO.println &(fmt "%d" FOO))))
```

このテクニックを使えば、Carp コードから参照したい暫定的な定義を追加できます。ヘルパー関数やマクロ、プリプロセッサディレクティブが長く複雑になる場合は、別の `h` ファイルにまとめて記述し、Carp ソースから `relative-include` する方法も検討してください。

### 型の登録

Carp では C で定義された型をいくつかの方法で登録できます。最も基本的なのは `register-type` 関数を利用する方法です。シンボルだけを渡すと、C 側の型を同名で Carp に登録します。例えば次のコードでは、C の `A` 型が Carp の `A` 型として登録されます。

```c
typedef int A;
```

```clojure
(register-type A)
```

`register-type` を呼ぶと、型名を書ける場所なら Carp のどこでも `A` を利用できるようになります。例えば関数シグネチャに含めることができます。

```clojure
(sig a-prn (Fn [A] String))
```

ただし、この段階では Carp は型名を知っているだけで、実装や値の生成方法は把握していません。値の構築は C 側のコードに依存します。

Carp からこの型の値を生成したい場合には次の 2 通りの手段があります。

1. C 側で初期化関数を定義し、Carp から `register` して利用する。
2. `register-type` にフィールドの配列を渡し、Carp に初期化子やアクセサを自動生成させる。

最初の方法では、C で定義した初期化関数を `register` すれば Carp から呼び出せます。

```c
typedef int A;

A initializer() {
  return 0;
}
```

```clojure
(register-type A)
(register initializer (Fn [] A))
;; returns a value of type A
(initializer)
```

2 つ目の方法として、`register-type` の呼び出しで空でないフィールド配列を渡すと、Carp が外部型向けの初期化子、ゲッター、セッター、表示用関数を自動生成します。生成された初期化子は指定したフィールドだけを初期化します。フィールドを省略したり名前を誤ったりすると、生成された初期化子がエラーの原因になります。

```clojure
(register-type B [])
:i B
=> B : Type
     init : (Fn [] B)
     prn  : (Fn [(Ref B q)] String)
     str  : (Fn [(Ref B q)] String)
}
(register-type C [x Int])
:i C
=> C : Type
   C : Module {
     init     : (Fn [Int] C)
     prn      : (Fn [(Ref C q) String])
     str      : (Fn [(Ref C q) String])
     set-x    : (Fn [C, Int] C)
     set-x!   : (Fn [(Ref C q), Int] ())
     update-x : (Fn [C, (Ref (Fn [Int] Int) q)] C)
     x        : (Fn [(Ref C q)] (Ref Int q))
}
```

さらに、`prn` および `str` 関数も自動的に対応するインタフェースを実装します。

ただし注意点があります。Carp は *外部型に紐づくメモリをデフォルトでは管理しません*。Carp で定義された型とは異なり、登録した型に対しては `copy` や `delete` の関数が自動生成されません。初期化子だけを利用しても、値に紐づくメモリ管理は自分で行う必要があります。Carp にメモリ管理を任せたい場合は `copy` と `delete` のインタフェースを実装してください。

必要に応じて、追加の文字列引数を指定することで登録型に対して Carp が出力する名前を上書きできます。C 側の型名が Lisp や Carp の命名規則に従っていない場合、例えば先頭が小文字の場合などに便利です。

```clojure
;; Emitted in C code as "A"
(register-type A)
;; Emitted in C code a "a_type"
(register-type A "a_type")
;; Emitted in C code as "b_type"
(register-type B "b_type" [x Int])
```

## コールバック

コールバックを利用する C API もあります。例として、コールバックと引数を受け取り、その関数を呼び出した結果を返す C 関数を定義してみます。

```clojure
(deftemplate runner (Fn [(Ptr ()) (Ptr ())] a)
                    "$a $NAME(void* fnptr, void* args)"
                    "$DECL {
                       return (($a(*)(void*))fnptr)(args);
                    }")

; Using a lambda capturing variables from its environment
(let [x 20 y 22 fnfn (fn [] (+ @&x @&y))]
  (= (runner (Function.unsafe-ptr &fnfn) (Function.unsafe-env-ptr &fnfn))
     42))

; Using a static function
(defn double [x] (Int.* @x 2))

(let [x 42]
  (= (runner (Function.unsafe-ptr &double) (Unsafe.coerce &x))
     84))
```

最初の例では環境をキャプチャするラムダを使用しています。`Function.unsafe-ptr` を使うと関数への `void*` が得られます。環境をキャプチャするラムダの場合、関数の第1引数が環境になるため、`Function.unsafe-env-ptr` で環境も渡します。

2 つ目の例では静的な関数を使っており、ここでも `Function.unsafe-ptr` を利用します。渡す引数は `Ref` から `(Ptr ())` にキャストする必要があります。

これらの操作ではすべて `void*` を扱うため型安全性は失われます。安全性を保証する責任は呼び出し側にあります。また、ポインタの寿命が対応する関数や環境の寿命を超えないように注意してください。

## Headerparse

`headerparse` は C ヘッダを解析し、`register` と `register-type` を自動生成してくれる Haskell スクリプトです。Carp のリポジトリでは `./headerparse` フォルダにあり、次のように利用します。

```sh
stack runhaskell ./headerparse/Main.hs -- ../path/to/c/header.h
```

利用できるフラグは次のとおりです。

* `[-p|--prefixtoremove thePrefix]` C 側識別子から特定の接頭辞を除去します。
* `[-f|--kebabcase]` 識別子をケバブケースに変換します。
* `[-c|--emitcname]` バインディングごとに C 側の識別子名を必ず出力します。

### 例

次の C ヘッダに対してスクリプトを実行してみます。

```sh
stack runhaskell ./headerparse/Main.hs -- -p "MyModule_" -f ../path/to/aheader.h
```

```c
// aheader.h
bool MyModule_runThisFile(const char *file);
```

すると以下が出力されます。

```clojure
(register run-this-file (λ [(Ptr CChar)] Bool) "MyModule_runThisFile")
```
