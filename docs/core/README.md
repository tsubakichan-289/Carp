Carp 標準ライブラリのリファレンスは [オンライン](http://carp-lang.github.io/carp-docs/core/core_index.html) で参照できますが、お使いの Carp 環境にもこのディレクトリにローカルコピーが含まれているはずです。

ローカルコピーを更新したい場合は、[簡単な Carp プログラム](./generate_core_docs.carp) を実行してください。実行時は `./docs/core` ではなく Carp のインストールディレクトリで次のコマンドを実行する点に注意してください。
```
carp -x ./docs/core/generate_core_docs.carp
```
