# コードの実行方法

このドキュメントは Carp を始めたばかりの方、特に[サンプル集](../examples)を試してみたい方向けのガイドです。

## 前提条件

まずは[Carp コンパイラと依存パッケージをインストール](Install.md)し、エラーメッセージなく起動できることを確認してください。

期待する出力例は以下の通りです。

```text
$ carp
Welcome to Carp X.Y.Z
This is free software with ABSOLUTELY NO WARRANTY.
Evaluate (help) for more information.
鲤
```

最後の行に表示されている `鲤` は [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) のプロンプトで、Carp が入力待ちであることを表します。

## REPL からコードを実行する

REPL では次のようにファイルをロードできます。

```bash
鲤 (load "some_file.carp")
```

ファイルパスは `carp` を起動したディレクトリからの相対パスで指定します（あるいは、そのファイルが[検索パス](Libraries.md)上に存在している必要があります）。複数のトップレベル式を含むコードをそのまま貼り付けても構いません。

ビルドと実行はそれぞれ以下のコマンドで行います。

```bash
鲤 (build)
```

続けて実行します。

```
鲤 (run)
```

## 端末からコードを実行する

REPL ではなく、従来の「コンパイルして実行する」フローで使いたい場合は次のようにします。

```bash
$ carp some_file.carp -x
```

`carp` に引数として指定したファイルは起動時に読み込まれます（REPL を起動するときも同様です）。`-x` フラグは「コンパイルしてすぐ実行し、そのまま終了する」という意味です。

実行ファイルを生成するだけでよい場合は `-b` を指定します。

```bash
$ carp some_file.carp -b
```
