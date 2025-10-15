## ライブラリとモジュールの扱い

Carp には Core という標準ライブラリが付属しており、ソースコードは[こちら](../core/)にあります。複数のモジュールで構成されており、多くは[オンライン](http://carp-lang.github.io/carp-docs/core/core_index.html)および[ローカル](./core/core_index.html)の両方でドキュメントを参照できます。ほとんどのモジュールはデフォルトで読み込まれます。詳細は [Core.carp](../core/Core.carp) を参照してください。

`CARP_DIR` 環境変数を[適切に設定](Install.md#carp_dir-の設定)していれば、残りのライブラリは `load` コマンドで簡単に読み込めます。たとえば Bench モジュールを使う場合は次のとおりです。

```clojure
(load "Bench.carp")
```

Bench モジュールの関数を呼ぶ際はモジュール名を付けて `(Bench.bench fib)` のように記述します。これを避けたい場合は `(use Bench)` も合わせて実行してください。

モジュールに含まれる関数を調べるには `info` コマンドを使います。

```clojure
(info Bench)
```

外部ライブラリを読み込む際は、ファイルシステム上のパスを相対／絶対で指定できます。より汎用的に使えるようにするには、Carp プロジェクトの検索パス（REPL で `:p` を実行すると確認できます）へ追加します。たとえば NCurses ライブラリを `(load "NCurses.carp")` で読み込めるようにするには次の設定を行います。

```clojure
(Project.config "search-path" "~/Projects/carp-ncurses")
```

この設定行を [profile.carp](Manual.md#profilesettings) に記述しておけば、すべてのプロジェクトで有効になります。

## Git 経由で読み込む

ライブラリは Git から直接読み込むこともできます。

```clojure
(load "git@github.com:hellerve/anima.carp@master")
```

すると [Anima](https://github.com/hellerve/anima) ライブラリが `~/.carp/libs/<library>/<tag>` にダウンロードされ、`anima.carp` が読み込まれます。安定版を利用したい場合は `@master` ではなくタグを指定してください。

ライブラリを読み込めるようにするには、ライブラリ名と同じファイル（上記の例では `anima.carp`）か `main.carp` を用意してエントリポイントにしてください。

非公開リポジトリでは SSH 経由の読み込みのみサポートされています。公開リポジトリなら HTTPS も利用できます。

```clojure
(load "https://github.com/hellerve/anima@master")
```

## ドキュメント生成

REPL で `save-docs` を実行すると、指定したモジュールの HTML ドキュメントを生成できます。

```clojure
> (save-docs Int Float String)
```

標準ライブラリ全体のドキュメントを更新する手順は [core/README.md](./core/README.md) に、ドキュメントジェネレーターの設定例は [generate_core_docs.carp](./core/generate_core_docs.carp) に記載されています。

## 自動生成された API ドキュメント

* [オンライン版標準ライブラリドキュメント](http://carp-lang.github.io/carp-docs/core/core_index.html)
* [ローカル版標準ライブラリドキュメント](./core/core_index.html)
* [SDL ライブラリドキュメント](http://carp-lang.github.io/carp-docs/sdl/SDL_index.html)

## 外部ライブラリの例
* [Anima](https://github.com/hellerve/anima) — シンプルな描画・アニメーションフレームワーク
* [Stdint](https://github.com/hellerve/stdint) — `stdint.h` で定義された型のラッパー
* [Socket](https://github.com/hellerve/socket) — C ソケットのラッパー
* [Physics](https://github.com/hellerve/physics) — phys.js の移植
* [NCurses](https://github.com/eriksvedang/carp-ncurses) — [ncurses](https://www.gnu.org/software/ncurses/)
* [Curl](https://github.com/eriksvedang/carp-curl) — Curl ライブラリへのシンプルなバインディング

Carp のパッケージ一覧は [Carpentry](https://github.com/carpentry-org) を参照してください。

あなたのライブラリも紹介したいですか？ ぜひ PR を送ってください！
