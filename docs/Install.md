# インストール

## 最新リリース

最新リリースは [https://github.com/carp-lang/Carp/releases](https://github.com/carp-lang/Carp/releases) を参照してください。

## ソースから Carp をビルドする

1. 最新の [Stack](https://docs.haskellstack.org/en/stable/README/) をインストールします。
2. このリポジトリをローカルにクローンします。
3. プロジェクトのルートディレクトリで `stack build` を実行します。
4. `stack install` を実行すると Carp のコマンドラインツールがシステム上にインストールされます。
5. Stack が実行ファイルを配置するディレクトリが PATH に含まれていることを確認します。例: `export PATH=~/.local/bin:$PATH`

## CARP_DIR を設定する

どこからでも `carp` を実行できるようにするには、実行ファイルがコアライブラリなどを見つけられる必要があります。環境変数 `CARP_DIR` を Carp リポジトリのルートに設定してください。

例として `.bashrc` などに以下を追加します。

```bash
export CARP_DIR=~/Carp/
```

設定後は任意の場所から Carp を起動できます。

```bash
$ carp
```

## POSIX 環境での UTF-8 対応 LC_CTYPE の設定

`carp` の対話型 REPL で UTF-8 を正しく扱うには（生成済みバイナリ自体は常に UTF-8 対応です）、Linux や macOS、あるいは Windows 10 上の Emacs eshell など POSIX 互換シェルで `LC_CTYPE` を UTF-8 対応の値に設定・エクスポートしておく必要があります。

例として `.bashrc` に以下を追加します。
```bash
export LC_CTYPE=C.UTF-8
```

ただし `LC_ALL` が設定されている場合は `LC_CTYPE` よりも優先されるため、`LC_ALL` を `unset` するか、同じく UTF-8 対応の値に設定してください。現在の `LC_ALL` と `LC_CTYPE` の値は `locale` コマンドで確認できます。

## C コンパイラ

`carp` 実行時には `main.c` という 1 つの C コードファイルが生成され、外部の C コンパイラでビルドされます。macOS と Linux ではデフォルトで `clang` を利用するので、インストールしておいてください（macOS では Xcode と開発者ツールを導入するのが簡単です）。

Windows では既定で `clang-cl.exe` を使用し、Clang でコンパイルした後 Visual Studio のリンカでリンクします。パッケージマネージャー [Scoop](https://scoop.sh/) を使うと LLVM のインストールが簡単です。また、Visual Studio に C/C++ 拡張を入れておく必要があります。Carp を使うのに WSL（Windows Subsystem for Linux）は不要です。

別のコンパイラやコマンドを使いたい場合は、次のようにビルドコマンドを指定できます。

```clojure
(Project.config "compiler" "gcc --important-flag")
```

## SDL や GLFW など

グラフィックス／サウンド／インタラクション関連のサンプルを動かすには、次のライブラリをインストールしておく必要があります。

* [SDL 2](https://www.libsdl.org/download-2.0.php) — クロスプラットフォームなゲーム／インタラクションライブラリ
* [SDL_image 2](https://www.libsdl.org/projects/SDL_image/) — 画像読み込みヘルパ
* [SDL_ttf 2](https://www.libsdl.org/projects/SDL_ttf/) — フォント描画
* [SDL_mixer 2](https://www.libsdl.org/projects/SDL_mixer/) — オーディオ再生
* [glfw](http://www.glfw.org) — OpenGL/Vulkan のレンダリングコンテキストを生成

macOS や Linux では [pkg-config](https://www.freedesktop.org/wiki/Software/pkg-config/) を使ってインクルードパスやリンクフラグを解決するので、外部ライブラリが見つかるよう適切に設定してください。

バインディングの利用で困ったことがあればぜひ知らせてください。依存関係の扱いではどうしても例外ケースが生じることがあります。問題報告や質問は[公式 Gitter チャンネル](https://gitter.im/carp-lang/Carp)でも歓迎しています。

## Windows 向け補足

mingw64 を使って clang をインストールすることもできますが、その場合はシェルを開くたびに `vcvarsall.bat amd64` もしくは `vcvarsall.bat x86` を実行し、必要なヘッダを見つけられるようにすることをおすすめします。詳しくは https://github.com/carp-lang/Carp/issues/700 や https://github.com/carp-lang/Carp/issues/1323 を参照してください。

また Windows で Carp を使う場合は、次の点にも注意してください。
* ソースファイルのエンコーディングは ANSI もしくは UTF-8 にする（UTF-8 BOM 付きは不可）。
* 改行コードは Unix (LF) か Windows (CR LF) を使用する（Macintosh (CR) だけの改行は不可）。
