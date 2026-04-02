---
title: "Biome + GritQL で FSD を守る"
emoji: "🍰"
type: "tech"
topics: ["biome", "fsd", "gritql", "typescript"]
published: false
publication_name: "chot"
---

## はじめに

[Feature-Sliced Design（FSD）](https://feature-sliced.design)のレイヤー依存ルール、人手で守るの辛くないですか？

FSD (v2) は 6 つのレイヤー間で**依存を上から下への一方向のみ**に制限するアーキテクチャですが、このルールは「気をつける」だけではいずれ破綻します。リンターで自動強制したいところです。

FSD のルール強制ツールには主に 3 つの選択肢があります。

- **ESLint** (`eslint-plugin-boundaries` 等) - おそらく最も普及しています。`no-restricted-imports` やプラグインベースです。
- **Steiger** - FSD 公式のアーキテクチャリンターです。コードの中身（AST）ではなくファイル構造やレイヤーの命名規則をチェックします。ESLint とは守備範囲が異なるので併用が前提です。LSP はサポートされていません。
- **Biome GritQL** - Biome 2.0 から使えるカスタムプラグインです。**この記事の主題**です。

私はリンター/フォーマッターに Biome を採用した Next.js プロジェクトで FSD を導入したいと考えていました。ESLint であれば `eslint-plugin-boundaries` 等で FSD のルール強制が可能ですが、Biome にはアーキテクチャレイヤーの依存関係を制御するプラグインが存在しません。そこで、Biome 2.0 で導入された GritQL カスタムプラグインを使い、同等のルールをフルスクラッチで実装しました。

この記事では、実際に書いた GritQL プラグインの設計と実装についてまとめています。ルールを含むテンプレートリポジトリはこちらです。

https://github.com/udonc/next-fsd-template

## GritQL とは

[GritQL](https://docs.grit.io/language/overview) は AST（抽象構文木）ベースのパターンマッチング言語です。Biome 2.0 から、`.grit` ファイルとして独自の lint ルールを定義できるようになりました。

基本構文はシンプルです。

```grit
language js

// AST ノードにマッチし、条件を満たしたら診断を出す
JsImport() as $stmt where {
  $stmt <: contains `$src` where { $src <: r"\"forbidden\"" },
  register_diagnostic(span=$src, message="このインポートは禁止です", severity="error")
}
```

- `language js` - 対象言語を宣言
- `JsImport()` - Biome の AST ノード名でパターンマッチ
- `$変数` - キャプチャ変数
- `r"..."` - 正規表現マッチ
- `register_diagnostic()` - エラーを報告

## GritQL で実装したルール - 全体像

FSD のルール強制は **Biome 組み込みの `noRestrictedImports`** と **GritQL カスタムプラグイン** の二層構造で実現しています。GritQL で書いた FSD 関連のルールは 3 つです。

| ルール | やりたいこと | GritQL が必要な理由 |
|--------|------------|-------------------|
| `fsd-no-cross-import` | `@features/cart/*` から `@features/auth` を import したらエラー。`typeof import()` や `../` による境界越えも検出 | 「今いるファイルが `features/cart/` なら `@features/auth` は NG」のように、**ファイルの位置に応じた動的判定**が必要 |
| `fsd-barrel-export-only` | `index.ts` に `export { Button } from "./ui/button"` 以外の記述（ロジックや import 文など）があったらエラー | **AST 構造の制約**（re-export 以外の構文を禁止）が必要 |
| `fsd-cross-import-api` | `@x/` ファイルに re-export 以外を書いたらエラー。entities 以外のレイヤーで `@x/` を使ってもエラー | 同上 + レイヤー限定の制約 |

これに加えて、深い相対パスの禁止（`no-deep-relative-import`）、テンプレートリテラルによる動的インポートの禁止（`no-template-literal-import`）、`import = require()` の禁止（`no-import-equals`）といった補助的なルールも GritQL で実装しています。これらは FSD 固有ではありませんが、静的解析の前提を守るために役立ちます。

一方、**レイヤー間の依存方向**（例: `features` → `widgets` の禁止）や**ディープインポートの禁止**は、`noRestrictedImports` の `overrides` で十分に表現できるため、GritQL は使わず適材適所で使い分けています。

## noRestrictedImports overrides との役割分担

`biome.jsonc` には 12 個の `overrides` ブロックがあり、段階的にインポート制限を定義しています。

### レイヤー別の段階的制限

```jsonc
// entities 層: shared のみ許可（他の全レイヤーを禁止）
{
  "includes": ["src/entities/**"],
  "linter": {
    "rules": {
      "style": {
        "noRestrictedImports": {
          "level": "error",
          "options": {
            "patterns": [
              { "group": ["@app", "@app/**"], "message": "entities → app は禁止" },
              { "group": ["@pages", "@pages/**"], "message": "entities → pages は禁止" },
              { "group": ["@widgets", "@widgets/**"], "message": "entities → widgets は禁止" },
              { "group": ["@features", "@features/**"], "message": "entities → features は禁止" },
              // ディープインポート禁止（@x/ は例外として許可）
              {
                "group": [
                  "@entities/*/**",
                  "!@entities/*/index.server",
                  "!@entities/*/index.client",
                  "!@entities/*/@x/*"
                ],
                "message": "ディープインポート禁止"
              }
            ]
          }
        }
      }
    }
  }
}
```

`!` プレフィックスによる否定パターンで、`index.server` / `index.client`（Server/Client バレル分離）や `@x/`（クロスインポート API）を例外として許可しています。

### バレルファイル専用のスコープ

```jsonc
{
  "includes": ["src/features/*/index.ts", "src/features/*/index.server.ts", ...],
  "linter": {
    "rules": {
      "style": {
        "noRestrictedImports": {
          "options": {
            "patterns": [
              // 全パスエイリアスを禁止（相対パスのみ許可）
              { "group": ["@app", "@app/**", "@pages", "@pages/**", ...] },
              // 許可セグメント以外の相対インポートを禁止
              { "group": ["./*", "./**", "!./ui", "!./ui/**", "!./model", "!./model/**", ...] }
            ]
          }
        }
      }
    }
  }
}
```

バレルファイルでパスエイリアスを禁止し、`./ui`, `./model`, `./api`, `./lib`, `./config` からの相対 re-export のみを許可します。

### GritQL が必要になる境界線

`noRestrictedImports` でカバーできるのは**静的なパスパターンの許可/禁止**までです。以下は GritQL でないと対応できません。

- **ファイルパスに基づく動的判定** - 「自分と同じレイヤーの別スライスか」はファイルの位置で決まる
- **AST 構造の制約** - 「re-export 以外の構文が含まれているか」はパスパターンでは判定できない
- **`typeof import()` の制約** - `noRestrictedImports` はこの構文を検知しない

この境界線を踏まえた上で、ここからは GritQL で書いた各ルールの中身に入ります。

## fsd-no-cross-import

FSD の[レイヤー間依存ルール](https://feature-sliced.design/ja/docs/reference/layers)と[クロスインポートの制約](https://feature-sliced.design/ja/docs/reference/public-api#public-api-for-cross-imports)に基づくルールで、一番複雑です。1 ファイルで **5 つの違反パターン** を検出します。

### 検出する 5 パターン

**(A) 同一レイヤー内クロスインポート**

```ts
// src/features/cart/ui/cart-page.tsx から
import { useAuth } from "@features/auth"; // NG: features 同士
```

FSD では同一レイヤーの別スライスを直接インポートできません。ただし、entities レイヤーのみ `@x/` クロスインポート API 経由でのアクセスを許可します。

**(B) 自スライスの `@x/` アクセス**

```ts
// src/entities/song/model/song-list.tsx から
import type { Song } from "@entities/song/@x/artist"; // NG: 自分自身の @x/ は不要
```

`@x/` は他スライスに公開するための API であり、自スライス内ではローカルセグメントから直接インポートすべきです。

**(C) `typeof import()` による上位レイヤー依存**

```ts
// src/entities/order/model/order.ts 内
type Props = typeof import("@features/auth"); // NG: entities → features
```

通常のインポート文は `noRestrictedImports` で制限できますが、`typeof import()` は型レベルの構文なので別途 GritQL で検出する必要があります。

**(D) `typeof import()` のディープインポート**

```ts
type Mod = typeof import("@features/auth/model/use-auth"); // NG: ディープインポート
```

**(E) 相対パス境界越え**

```ts
// src/features/cart/ui/cart-page.tsx から
import { something } from ".."; // NG: スライス/セグメント境界を越える
```

### 実行結果の例

パターン (A) に違反すると、`biome check` でこんなエラーが出ます。

```
src/features/cart/ui/cart-page.tsx:1:27 plugin ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  × FSD違反: 同一レイヤー内の別スライスからインポートできません。
    @x/ クロスインポートAPIを使用してください。

  > 1 │ import { useAuth } from "@features/auth";
      │                         ^^^^^^^^^^^^^^^^
    2 │ export { useAuth };
```

### 実装の工夫: `$filename` による動的判定

パターン (A) が `noRestrictedImports` では実現できない理由は、**「自分がどのレイヤー・スライスにいるか」をファイルパスから動的に判定する必要がある**からです。

```grit
// (A) 同一レイヤー内クロスインポート
{
  // インポート先のレイヤーとスライスを正規表現でキャプチャ
  $src <: r"^\"@(pages|widgets|features|entities)/([^/\"]+)(?:/.*)?\"$"($src_layer, $target),
  // 現在のファイルのレイヤーとスライスを $filename からキャプチャ
  $filename <: r".*/(pages|widgets|features|entities)/([^/]+)/.*"($layer, $current),
  // 同一レイヤー かつ 別スライス → 違反
  $src_layer <: $layer,
  not { $target <: $current },
  // @x/ クロスインポートAPI例外
  not {
    $layer <: "entities",
    $src <: r"^\"@entities/[^/]+/@x/([^/\"]+)\"$"($consumer),
    $consumer <: $current
  },
  register_diagnostic(
    span=$src,
    message="FSD違反: 同一レイヤー内の別スライスからインポートできません。@x/ クロスインポートAPIを使用してください。",
    severity="error"
  )
}
```

GritQL の `$filename` は、現在解析中のファイルのパスが自動的に束縛される特殊変数です。これにより「`features/cart/` にいるファイルが `@features/auth` をインポートしている」のような、ファイル位置に依存した判定ができます。

### パフォーマンス最適化: `contains` 走査の統合

GritQL の `contains` は AST のサブツリーを再帰的に走査するため、コストが高い操作です。単純に 5 パターンを独立して書くと最大 5 回の `contains` が発生します。

このルールでは、関連するパターンを 2 つのブロックに統合しています。

```grit
or {
  // ── 統合ブロック ABE: import/export 文 ──
  or {
    JsImport() as $stmt,
    JsExportNamedFromClause() as $stmt,
    JsExportFromClause() as $stmt,
    JsImportCallExpression() as $stmt,
    TsImportType() as $stmt
  } where {
    $stmt <: contains `$src` where { $src <: r"^\"(?:@|\.\.).*" },
    or {
      // (A) クロスインポート ... 前述の判定ロジック
      // (B) 自スライス @x/ ... $provider <: $current で検出
      // (E) 相対パス境界越え ... $filename の深さで境界判定
    }
  },
  // ── 統合ブロック CD: typeof import() ──
  TsImportType() as $stmt where {
    $stmt <: contains `$src` where { $src <: r"^\"@.*" },
    or {
      // (C) 上位レイヤー依存 ... $filename のレイヤーと $src のレイヤーを比較
      // (D) ディープインポート ... パスの深さで検出
    }
  }
}
```

**ブロック ABE** は `@` または `..` で始まるインポートパスを持つ文を 1 回の `contains` で拾い、その中で 3 パターンに分岐します。**ブロック CD** は `typeof import()` に特化し、パスエイリアスで始まるもののみを 1 回の `contains` で処理します。結果、最大 5 回から最大 2 回に削減しています。

### パターン (C) の実装: レイヤー依存の全組み合わせ

`typeof import()` に対するレイヤー依存チェック（パターン C）は、各レイヤーの許可/禁止を愚直に列挙しています。

```grit
// (C) 上位レイヤー依存
or {
  { $filename <: r".*/src/shared/.*",   $src <: r"^\"@(?:app|pages|widgets|features|entities)(?:/.*)?\"$" },
  { $filename <: r".*/src/entities/.*", $src <: r"^\"@(?:app|pages|widgets|features)(?:/.*)?\"$" },
  { $filename <: r".*/src/features/.*", $src <: r"^\"@(?:app|pages|widgets)(?:/.*)?\"$" },
  { $filename <: r".*/src/widgets/.*",  $src <: r"^\"@(?:app|pages)(?:/.*)?\"$" },
  { $filename <: r".*/src/pages/.*",    $src <: r"^\"@app(?:/.*)?\"$" }
},
register_diagnostic(...)
```

`shared` は全レイヤーへの依存が禁止され、`entities` は `shared` 以外の全レイヤーへの依存が禁止され、…と段階的に制限が緩くなります。通常のインポート文には `noRestrictedImports` の overrides で同じ制約を掛けているので、ここでは `typeof import()` のみをカバーする形です。

## fsd-barrel-export-only と fsd-cross-import-api

続いて、ファイル構造に対する制約です。

### fsd-barrel-export-only

FSD ではスライスの[公開 API](https://feature-sliced.design/ja/docs/reference/public-api) をバレルファイル（`index.ts`）で定義します。このバレルには re-export 以外の記述を許可しません。

```grit
file($name, $body) where {
  // スライスのバレル or shared モジュールのバレルにマッチ
  // shared は shared/<segment>/<module>/index.ts と2階層深い構造になっている
  $name <: r".*/src/(?:(?:pages|widgets|features|entities)/[^/]+|shared/[^/]+/[^/]+)/index\.[^/]+$",
  $body <: contains bubble or {
    JsDirective() as $stmt,
    JsImport() as $stmt,
    JsVariableStatement() as $stmt,
    JsVariableDeclarationClause() as $stmt,
    JsFunctionDeclaration() as $stmt,
    JsClassDeclaration() as $stmt,
    // ... 他にも TsTypeAliasDeclaration, TsEnumDeclaration 等を列挙
    JsExportDefaultExpressionClause() as $stmt,
    JsExportDefaultDeclarationClause() as $stmt
  } where {
    register_diagnostic(
      span=$stmt,
      message="FSD違反: バレルファイル (index.ts) では re-export 以外の記述はできません。",
      severity="error"
    )
  }
}
```

ポイントは **「re-export 以外を禁止する」** という設計です。GritQL には「この構文以外」を直接表現する否定マッチがないため、禁止したい構文（`JsFunctionDeclaration` や `JsClassDeclaration` など）を全て列挙して `contains` で探します。結果として、列挙されていない `JsExportNamedFromClause` や `JsExportFromClause`（re-export 構文）だけが許可される形になります。

`file($name, $body)` コンテキストを使うことで、ファイル名パターンに基づいてルール適用範囲を限定しています。

### fsd-cross-import-api

`@x/` ディレクトリは、entities レイヤー内でスライス間の[クロスインポート](https://feature-sliced.design/ja/docs/reference/public-api#public-api-for-cross-imports)を制御するための仕組みで、FSD の Public API として公式に定義されています。

```
src/entities/
├── song/
│   ├── model/song.ts
│   ├── @x/
│   │   └── artist.ts    ← artist スライス向けに公開する型を re-export
│   └── index.ts
└── artist/
    └── ...               ← import type { Song } from "@entities/song/@x/artist"
```

このルールは 2 つの制約を検出します。

1. **entities の `@x/` ファイルは re-export のみ** - `fsd-barrel-export-only` と同じホワイトリスト方式で、ロジックやインポートの記述を禁止
2. **entities 以外のレイヤーでは `@x/` を使用禁止** - `$name` の正規表現でレイヤーを判定し、re-export を含む全ての記述を禁止（ファイルの存在自体が NG）

## セットアップ方法

GritQL プラグインの導入は簡単です。

### 1. `.grit` ファイルを配置

プロジェクトルートに任意のディレクトリ（例: `biome/`）を作成し、`.grit` ファイルを配置します。

```
your-project/
├── biome/
│   ├── fsd-no-cross-import.grit
│   ├── fsd-barrel-export-only.grit
│   └── ...
├── biome.jsonc
└── ...
```

### 2. `biome.jsonc` にプラグインを登録

```jsonc
{
  "plugins": [
    "./biome/fsd-no-cross-import.grit",
    "./biome/fsd-barrel-export-only.grit",
    "./biome/fsd-cross-import-api.grit",
    "./biome/no-deep-relative-import.grit",
    "./biome/no-template-literal-import.grit",
    "./biome/no-import-equals.grit"
  ]
}
```

### 3. 実行

```bash
biome check src/
```

これだけで GritQL ルールが有効になります。`noRestrictedImports` の overrides と組み合わせる場合は、テンプレートリポジトリの [`biome.jsonc`](https://github.com/udonc/next-fsd-template/blob/main/biome.jsonc) を参考にしてください。

## GritQL を使ってみてわかったこと

### 良い点

#### 宣言的で記述量が少ない

GritQL はパターンマッチと条件分岐だけでルールを表現できます。AST の走査ロジックや診断の報告処理を自分で書く必要がなく、「どんな構文を見つけたいか」を宣言するだけです。本記事の `fsd-barrel-export-only` は実質 20 行程度で、バレルファイルの構造制約を完結させています。

#### AST ノード名が明確

`JsImport`, `TsImportType`, `JsExportNamedFromClause` など、ノード名から役割が分かります。

#### 正規表現との組み合わせが強力

ファイルパスやインポートパスの構造的なマッチングが簡潔に書けます。

#### `$filename` でファイル位置に応じた動的判定ができる

GritQL の `$filename` は現在解析中のファイルパスが自動的に束縛される組み込み変数です。これにより「自分がどのレイヤー・スライスにいるか」をパスから判定し、ルールの挙動を動的に切り替えられます。本記事の `fsd-no-cross-import` はこの仕組みで成り立っています。

#### セットアップが手軽

`.grit` ファイルを置いて `biome.jsonc` の `plugins` に追加するだけで動きます。ビルドステップや追加パッケージのインストールは不要です。

#### Biome エコシステムに統合できる

ESLint を別途導入する必要がなく、`biome check` 一発でビルトインルールと GritQL ルールをまとめて実行できます。

### 注意点

#### ドキュメントが薄い

Biome の GritQL サポートはまだ発展途上で、使える AST ノード名の一覧が不十分です。`biome rage` の出力を頼りに探す場面がありました。ただ、Claude Code などの AI エージェントにリサーチを任せればそこまで苦にはなりません。

#### 書き方で実行速度が変わる

`contains` は AST のサブツリーを再帰的に走査するため、無造作に使うとパフォーマンスに影響します。本記事の `fsd-no-cross-import` では、5 パターンを 2 ブロックに統合して `contains` の走査回数を削減しました。GritQL は宣言的な見た目に反して、書き方の工夫がそのまま速度に反映されます。

#### ビルトインルールで表現できるなら GritQL を使わない方がいい

GritQL プラグインの実行はビルトインルールに比べると遅いです。本記事でもレイヤー間の依存方向やディープインポートの禁止は `noRestrictedImports` の overrides に任せ、GritQL はビルトインでは不可能なルールだけに絞っています。

#### デバッグが難しい

「なぜマッチしないのか」を調べる手段が限られます。`register_diagnostic` を仮置きして段階的に条件を絞り込むしかありません。

#### エディタ支援が薄い

`.grit` ファイルに対する補完は現時点ではほぼありません。シンタックスハイライトは [tree-sitter-gritql](https://github.com/getgrit/tree-sitter-gritql) が公開されているので、Neovim 等では自前でハイライト設定を組むことで対応できました。

### ESLint カスタムルールとの比較

GritQL は宣言的な `.grit` ファイル 1 つで完結し、`biome.jsonc` の `plugins` にパスを追加するだけで動きます。ただし、ESLint のようなプラグインの配布・共有の仕組みは現時点では整っていません。npm でインストールして使うといったことはできないため、プロジェクト間で再利用するにはファイルをコピーするか、リポジトリを参照する必要があります。

FSD のルールは「パターンマッチ + 正規表現」で十分に表現できるため、GritQL との相性が良かったと言えます。

## まとめ

Biome の GritQL プラグインで FSD のレイヤー依存ルールを実装しました。`noRestrictedImports` の overrides で静的なパスパターン制約を、GritQL で動的・構造的な制約を担当する二層構造です。

GritQL はまだ発展途上ですが、「パターンマッチ + 正規表現で表現できるアーキテクチャルール」であれば、ESLint カスタムルールよりも少ないコード量で実装できます。FSD に限らず、独自のインポートルールやファイル構造の制約を lint で強制したい場面で有力な選択肢になるはずです。GritQL のエコシステムが成熟し、デバッグツールやエディタ支援が充実してくれば、さらに実用的になると期待しています。

テンプレートのソースコードはこちらです。

https://github.com/udonc/next-fsd-template
