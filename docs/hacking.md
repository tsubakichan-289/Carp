# Carp コンパイラをハックする

この文書では Carp コンパイラを変更する際のヒントやメモ、補足説明、実例などをまとめています。網羅的なガイドではなく、過去にコンパイラに手を入れた人たちが残した知見の寄せ集めだと思ってください。

> なお、コンパイラやコンパイル処理に関する基本的な用語には慣れているものとして話を進めます。

## 構成

Carp コンパイラのソースコードは `src/` ディレクトリにあります。ざっくり言うと、Carp は 4 つの主要なパス（フェーズ）で構成されています。

![carp compiler phases](./compiler-passes.svg)

各ソースファイルはコンパイラの 1 つ以上のフェーズに関わっています。以下の節では、それぞれのフェーズの役割と重要なソースファイルを簡単に紹介します。どのフェーズの挙動を変える必要があるのかあたりを付ける際の道標として活用してください。

> 補足: いくつかのファイルは複数フェーズで共通に利用されます。そのため、同じファイルが何度か登場する場合があります。

### パース

パースフェーズは `.carp` ソースコードを抽象構文木（AST）へ変換します。Carp では AST ノードを `XObj` という抽象データ型で表現します。`XObj` はコンパイラ全体で頻繁に登場し、さまざまなフェーズやコンテキストで利用されます。各 `XObj` は次の 3 つから構成されています。

- `Obj` : Carp のソースコードを抽象化したデータ表現
- `Info` : 生成元ソースコードに関する補足情報（ファイル上の位置など）
- `Ty` : [型システム](#type-system)が決定した `Obj` の Carp 型

主に関係するソースは以下の通りです。

- `Parsing.hs` — Carp のソースコードを AST へ変換する処理
- `XObj.hs` — Carp で有効な AST ノード定義

### 動的評価器

[マクロガイド](Macros.md#inner-workings)で説明されているとおり、動的評価器はコンパイラの中心的なコンポーネントです。名前の通り、`XObj` を評価して[コード生成](#code-emission)の準備を整えます。評価が行う主な処理は次のとおりです。

- マクロと動的関数の展開
- ほかのフォームへのバインディング解決
- フォームの型推論要求
- フォームの借用検査要求

評価器はコンパイル対象となる `XObj` 群に加えて `Context` を利用します。`Context` はコンパイラ全体の状態を保持するグローバルなオブジェクトです。内部には `Env` 型で表される複数の環境があり、それぞれが既知のバインディングを保持します。各フェーズは必要に応じて環境を参照し、フォームの評価・型解決・コード生成に備えます。

`Binder` も評価で重要な抽象データ型です。ソースコード内で名前に束縛されたあらゆる値は `binder` へと変換され、該当する `XObj` と追加メタデータを保持します。`Binder` は `Context` 内の環境へ登録されます。

評価に関係する主なソースは次の通りです。

- `Eval.hs` — 評価器のエントリーポイント
- `Obj.hs` — `Env` を束ねた `Context` の定義、`Env` や `Binders` の管理
- `Primitives.hs` — 引数を評価しない組み込み関数（キーワード）の定義
- `Commands.hs` — 引数を評価する組み込み関数（キーワード）の定義
- `StartingEnv.hs` — コンパイラ起動時に利用する初期環境の定義（全コマンド・プリミティブの登録）
- `Lookup.hs` — 指定した環境 (`Env`) から `Binder` を検索する関数
- `Expand.hs` — フォームを走査して構文解析を行う関数群（以前はマクロ展開も担当していたが、現在は `Eval.macroExpand` に移譲）
- `Infer.hs` — 型推論を行う関数群（型システムへの入り口）
- `Qualify.hs` — シンボルへ適切なモジュール名を付加

型システムや借用チェッカ関連のファイルも評価に深く関わりますが、ここではコア部分に絞っています。動的評価器はコンパイラ全体の「指揮者」と言える存在で、他のコンポーネントを連携させながらコンパイルを進行させます。

>動的評価器についてさらに詳しく知りたい場合は、[マクロガイドの内部構造の章](Macros.md#inner-workings)を参照してください。

### 型システム

型システムは Carp のフォームの型を検証し、プログラム全体の型安全性を保証します。また、多相性をサポートし、多相型を具体型へ置き換える役割も担います。Carp の型は `Ty` データ型で表現されます。

主なソースファイルは以下のとおりです。

- `Types.hs` — Carp における有効な型 (`Ty`) の定義。2 つの型が両立するかどうかを調べる単一化処理や、Carp の型名を C の識別子へ変換するマングリング処理も含まれます。
- `TypeError.hs` — 型チェック時のエラー定義
- `AssignTypes.hs` — 変数へ具体的な型を割り当て
- `Polymorphism.hs` — 多相関数を具体化した際に、具体的な C 側識別子を決定
- `Validate.hs` — ユーザ定義型の妥当性を検証
- `Constraints.hs` — 環境内の型や型変数のあいだにある制約を導出・解決
- `Concretize.hs` — 多相型を含むフォームを具体型へ変換
- `InitialTypes.hs` — 与えられた `XObj`（AST ノード）の初期型を推定
- `GenerateConstraints.hs` — 指定したフォームに関する型制約を算出

### 借用チェック／所有権システム

借用チェックとライフタイム引数は[型システム](#type-system)を拡張した仕組みです。そのため、型システムで重要なファイルは借用チェッカにとっても同じく重要です。

### コード生成

コンパイラの最後の仕事は、入力された Carp コードに対応する C コードを出力することです。コード生成は `Template` の概念に大きく依存しており、評価済みの Carp AST ノードに基づいて C の文字列を組み立てる仕組みになっています。

主なソースは次のとおりです。

- `ArrayTemplates.hs` — Carp の配列操作に対応する C コードのテンプレート
- `StaticArrayTemplates.hs` — StaticArray 操作用のテンプレート
- `Deftype.hs` — ユーザ定義構造体（積型）のテンプレートと関連処理
- `Sumtypes.hs` — ユーザ定義和型のテンプレートと関連処理
- `StructUtils.hs` — 構造体のユーティリティ関数向けテンプレート
- `Template.hs` — C コード生成に関する共通の手順
- `ToTemplate.hs` — C コード文字列からテンプレートを作成するヘルパ
- `Scoring.hs` — 型や `XObj` の情報を元に、生成した C のバインディングを適切な順序に並べ替える
- `Emit.hs` — 評価済みの Carp コードに基づいて最終的な C コードを出力

### その他のソース

上記以外にも、コンパイラ内でさまざまな役割を果たすユーティリティ的なファイルが存在します。

- `Repl.hs` — REPL の機能（キーワード補完、REPL コマンドなど）
- `Util.hs` — 汎用的なユーティリティ関数
- `ColorText.hs` — REPL やコンパイラ出力の色付けをサポート
- `Path.hs` — ファイルパス関連の処理
- `RenderDocs.hs` — アノテーション付き Carp コードからドキュメントを生成する機能

## ミニ HowTo 集

コンパイラへの変更のなかには頻度の高いものがあり、手順がある程度決まっています。以下では、代表的なケースの手順を紹介します。

### 新しいプリミティブを追加する

特別な事情がない限り、新しいプリミティブを追加する手順は次のとおりです。

1. `Primitives.hs` に新しいプリミティブを定義する
2. `StartingEnv.hs` で `makePrim` を使って初期環境へ登録する

#### プリミティブを定義する

プリミティブは `Primitive` 型の関数です。

```
type Primitive = XObj -> Context -> [XObj] -> IO (Context, Either EvalError XObj)
```

各プリミティブは自身を表す `XObj`、コンパイラの `Context`、そして引数となる `XObj` のリストを受け取ります。戻り値は処理結果を反映した新しい `Context` と、評価結果の `XObj` もしくはユーザへ報告する評価エラーのいずれかです。

例えば `defmodule` プリミティブと `Primitive` 型の対応は次のようになります。

```
(defmodule Foo (defn bar [] 1))
 |         |-----------------|
 XObj      [XObj] (arguments)
```

> `Context` 引数はコンパイラの状態を表すためのもので、Carp コード上に直接対応するものはありません。

`Primitives.hs` では、Carp コードから呼び出すシンボル名を `<name>` としたとき、プリミティブ名を `primitive<name>` とする命名規則に従ってください。`defmodule` の場合は `primitiveDefmodule` になります。

多くのプリミティブは次の 3 ステップで実装できます。

- 引数の `XObj` をパターンマッチする
- 現在の `Context` から既存のバインダを探索する
- 引数の `XObj` の種類に応じてロジックを実行し、必要に応じて `Context` を更新する

これらの手順を追いながら、例として `immutable` プリミティブを実装してみましょう。`immutable` は、`def` されたフォームに対して `/set!` を禁止するメタ情報を付与する機能だとします。

- ステップ 1. 引数のパターンマッチ

  Carp 側ではプリミティブを次のように呼び出します。

  ```
  (immutable my-var)
  ```

  つまり、引数は 1 つで、`Sym` でなければなりません。パターンマッチは次のように書けます。

  ```
  primitiveImmutable :: Primitive -- 新しいプリミティブ
  primitiveImmutable xobj ctx [XObj (Sym path@(SymPath) _)] =
    -- TODO: ここに実装を追加
  primitiveImmutable _ _ xobjs = -- 引数の個数や型が想定外ならエラー
    return $ evalError ctx ("`immutable` expected a single symbol argument, but got" ++ show xobjs) (info xobj)
  ```

- ステップ 2. 現在のコンテキストからバインダを探す

  正しい引数が渡されたと仮定し、`Sym` が `def` で束縛された変数なのかを調べます。

  `Lookup.hs` には `Context` 内の各環境からバインディングを検索する関数が定義されています。ここではシンボルに対応する `def` が存在するか調べ、存在しなければエラーにします。

  ```
  primitiveImmutable :: Primitive
  primitiveImmutable xobj ctx [XObj (Sym path@(SymPath) _)] =
    let global = contextGlobalEnv ctx
        binding = lookupInEnv path global
    in  case binding of
          Just (_, Binder meta (XObj )) -> -- TODO: def であることを確認できたら処理を続ける
          _ ->
            return $ evalError ctx ("`immutable` expects a variable as an argument") (info xobj)
  primitiveImmutable _ _ xobjs =
    return $ evalError ctx ("`immutable` expected a single symbol argument, but got" ++ show xobjs) (info xobj)
  ```

- ステップ 3. ロジックを実行し、`Context` を更新

  無事 `def` だと確認できたら、バインダの `MetaData` に `immutable = true` という項目を追加します。今後 `set!` を呼び出す際にこのメタ情報を参照すれば、書き換えを禁止できます。

  ```
  primitiveImmutable :: Primitive
  primitiveImmutable xobj ctx [XObj (Sym path@(SymPath) _)] =
    let global = contextGlobalEnv ctx
        binding = lookupInEnv path global
    in  case binding of
          Just (_, Binder meta def@(XObj Def _ _)) ->
            let oldMeta = getMeta meta
                newMeta = meta {getMeta = Map.insert "immutable" trueXObj oldMeta}
            in  return $ ctx {contextGlobalEnv = Env (envInsertAt global path (Binder newMeta def))}
          _ ->
            return $ evalError ctx ("`immutable` expects a variable as an argument") (info xobj)
  primitiveImmutable _ _ xobjs =
    return $ evalError ctx ("`immutable` expected a single symbol argument, but got" ++ show xobjs) (info xobj)
  ```

これでプリミティブ側のロジックは完成です。最後に `StartingEnv.hs` へ登録しましょう。

#### 初期環境へ登録する

初期環境へプリミティブを追加するには `makePrim` を呼び出します。

```
, makePrim "immutable" 1 "annotates a variable as immutable" "(immutable my-var)" primitiveImmutable
```

この実装はバインディングにメタデータを付与しただけなので、実際に `set!` を禁止するには `set!` 側のロジックで `immutable` メタ情報を参照する処理を追加する必要があります。
