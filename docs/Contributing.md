# コントリビューションガイド

Carp への貢献をご検討いただきありがとうございます。

本書は現時点では開発面でのコントリビューションに重点を置いていますが、あらゆる形での参加を歓迎しています。

## コミュニティ
まずは Carp の Gitter チャンネルに参加するのがおすすめです。  
[https://gitter.im/carp-lang/Carp](https://gitter.im/carp-lang/Carp)

## コンパイラの理解
Carp コンパイラの内部構造については [hacking.md](hacking.md) が穏やかな入門になっています。

## リポジトリへのコミット
Carp プロジェクトでは [Conventional Commits](https://www.conventionalcommits.org) を採用しています。コミットメッセージが規約に従っているかをチェックする `commit-msg` フックが用意されていますので、最初のコミット前に `./scripts/git-hooks/setup.sh` を実行し、フックを有効化しておいてください。

## ライセンス
Carp は現在 ASL 2.0 ライセンスの下で公開されています。
