## コンパイラマニュアル

### 関連ページ

* [インストール](Install.md) — Carp コンパイラの入手と設定
* [コードの実行方法](HowToRunCode.md) — .carp ファイルのコンパイルと実行
* [ツール](Tooling.md) — 対応エディタ
* [ライブラリ](Libraries.md) — ライブラリ／モジュールの扱い方
* [マルチメディア](Multimedia.md) — グラフィックスやサウンド関連
* [マクロ](Macros.md) — Carp のマクロシステムガイド
* [組み込み開発](Embedded.md) — 組み込み環境で Carp を使うためのヒント
* [用語解説](Terminology.md) — よく使う用語の意味

Carp の言語仕様や構文・セマンティクスについては [Carp Language Guide](LanguageGuide.md) を参照してください。

### REPL の基本

Carp は REPL と密接に統合されており、プログラムに対する操作はすべてここから行えます。

ディスク上のコードを読み込むには `(load "filename.carp")` を使います。これによりソースファイル `filename.carp` が現在の「プロジェクト」に追加されます。プロジェクトとは、REPL 上でソースファイルとコンパイラ設定を束ねる軽量な概念で、IDE（Eclipse や Visual Studio）上のプロジェクトに似ています。

現在のプロジェクトをビルドするには `(build)` を実行します。メイン関数が定義されていれば実行ファイル、そうでなければ動的ライブラリが生成されます。ライブラリを生成した場合はグローバル変数の初期化が自動では行われないため、利用側が C 関数 `carp_init_globals` もしくは Carp 関数 `System.carp-init-globals` を呼び出してください。

生成物は既定で `out` ディレクトリに保存されます。このほか、プロジェクトに関する各種設定はコマンドで変更できます。利用可能なコマンドは `(help "project")` で確認してください。

REPL には代表的な操作用ショートカットが用意されています。

```
:r   プロジェクト内のすべてのファイルを再読み込み
:b   プロジェクトをビルド
:x   実行ファイルが存在する場合に実行
:c   生成された C コードを表示
:e   関数束縛と型情報を含む環境を表示
:p   プロジェクト情報と設定を表示
:h   ヘルプを表示
:q   REPL を終了
```

### 他の Lisp の REPL との違い

Carp の REPL は強力ですが、一般的な Lisp の REPL とは異なる制約があります。式を入力して Enter を押すと、次のいずれかが起こります。

1. `defndynamic` で定義された関数（または組み込みコマンド）を呼び出した場合、その場で評価されます。これは伝統的な動的 Lisp のインタープリタに近い挙動です。動的関数はコンパイル済みコードでは利用できません。主な用途はマクロ内やビルド設定をプログラム的に制御する場面です。

2. `defn` で定義された関数を呼び出す場合は「通常の」 Carp 関数であり、C を経由してコンパイルされ、REPL の子プロセスとして実行されます。そのため、グローバル変数などプログラムの他の部分を変更することはできません。

3. トップレベルのフォームが関数呼び出しでない場合、REPL が混乱することがあります。たとえば Carp 関数の呼び出しを要素に持つ配列を入力すると、配列は動的に扱われますが関数呼び出しは評価されません。現時点での回避策は、`defn` で関数として包み、その関数を呼ぶことです。将来的には修正される予定です。

### アノテーションの追加

Carp には Clojure に着想を得た柔軟なメタデータシステムがあり、環境中のバインディングに任意の情報を付与・取得できます。一般的な操作は `(meta-set! <path> <key> <value>)` と `(meta <path> <key>)` です。

この仕組みの上にいくつか便利なマクロが実装されています。

```clojure
(doc <path> "This is a nice function.") ; ドキュメント文字列
(sig <path> (Fn [Int] Bool))            ; 型注釈
(private <path>)                        ; 他モジュールからアクセス不可にする
```

ここで `<path>` は `foo` や `Game.play` のようなシンボルです。

ドキュメント文字列から HTML を生成するには次のコマンドを実行します。

```clojure
(save-docs <module 1> <module 2> <etc>)
```

### バインディングの型を調べる

```clojure
鲮 (type <binding>)
鲮 :t <binding>
```

### モジュール内のバインディング一覧

```clojure
鲮 (info <module name>)
鲮 :i <module name>
```

### マクロの展開結果を確認する

```clojure
鲮 (expand 'yourcode)
鲮 :m yourcode
```

### プロジェクト設定

REPL での作業単位である「プロジェクト」は `(Project.config <setting> <value>)` で設定できます。主な設定項目は次の通りです。

* `"cflag"` — コンパイラに渡すフラグを追加
* `"libflag"` — リンク時に渡すライブラリフラグを追加
* `"pkgconfigflag"` — pkg-config 実行時に渡すフラグを追加
* `"compiler"` — `build` 時に使用するコンパイラを指定
* `"target"` — ターゲットトリプルを設定（クロスコンパイル向け）
* `"title"` — プロジェクト名を設定。生成されるバイナリ名に影響
* `"prompt"` — REPL のプロンプト文字列を設定
* `"search-path"` — Carp が `.carp` ファイルを探すパスを追加
* `"output-directory"` — ビルド生成物の出力先
* `"docs-directory"` — 生成したドキュメントの出力先
* `"generate-only"` — true にすると C コンパイラを呼ばず C コード生成のみを行う

* `"echo-c"` — `def` や `defn` の定義時に生成された C コードを表示
* `"echo-compiler-cmd"` — ビルド時に実行するコンパイラコマンドを表示
* `"print-ast"` — `info` コマンドで AST も表示

例: プロジェクトタイトルを設定する

```clojure
鲮 (Project.config "title" "Fishy")
```

別のコンパイラを使う
```clojure
鲮 (Project.config "compiler" "tcc")
```

### プロファイル設定

XDG の設定ディレクトリ `carp/` に `profile.carp` を置くと、コンパイラ起動後（コアライブラリ読み込み後、他のソースを読む前）に自動でロードされます。これは全プロジェクトで共通して使いたい設定やヘルパ関数を記述するのに便利です。

Windows では `C:/Users/USERNAME/AppData/Roaming/carp/profile.carp` に配置します。

### コンパイラフラグ

コマンドラインからコンパイラを呼び出す際は、以下のフラグで挙動を制御できます。

* `-b` — コードをビルドして終了
* `-x` — コードをビルドして実行し（メイン関数が必要）、終了
* `--no-core` — コアライブラリを読み込まずに実行
* `--log-memory` — 実行ファイルが malloc/free の呼び出しをすべてログ出力
* `--optimize` — 安全チェック（配列境界など）を無効化し、C コンパイラに `-O3` を付与
* `--check` — バイナリを出力せず、検出したエラーを（機械可読な形で）報告
* `--generate-only` — C ソースの生成のみに留め、コンパイルしない
* `--eval-preload` — コード読み込み前（ただし `profile.carp` 読み込み後）に指定した文字列を評価

### 式から生成された C コードを確認する

```clojure
鲮 (c '(+ 2 3))
```

### クロスコンパイル

クロスコンパイルは早い段階で有効化する必要があります。たとえば `profile.carp` に次を記述します。

```clojure
(Project.config "compiler" "zig cc --target=x86_64-windows-gnu")
(Project.config "target" "x86_64-windows-gnu")
```

または `--eval-preload` を使って次のように指定できます。

```sh
carp --eval-preload '(Project.config "compiler" "zig cc --target=x86_64-windows-gnu") (Project.config "target" "x86_64-windows-gnu")' -b whatever.carp
```
