# リリースチェックリスト

以下の手順をおおむね順番に実施します。

## 1. Cabal プロジェクトのバージョン更新

[CarpHask.cabal](../CarpHask.cabal) の 2 行目を更新します。

## 2. REPL の「Welcome to Carp X.Y.Z」メッセージ更新

[Main.hs](../App/Main.hs) を編集します。

## 3. README の冒頭文を更新

[README.md](../README.md) を参照してください。

## 4. 変更履歴の更新

[CHANGELOG.md](../CHANGELOG.md) を更新します。

## 5. master でコミットを作成

```bash
$ git add .
$ git commit -m "build: Release X.Y.Z"
```

## 6. タグを付けてプッシュ

```bash
$ git tag vX.Y.Z
$ git push --tags
```
