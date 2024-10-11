---
title: "ぼくがかんがえた最強の Visual Studio Code カスタマイズ 2024"
published: true
type: "tech"
topics: ["vscode", "editor"]
emoji: "✨"
---

もし世界が消滅して Visual Studio Code の設定が失われてしまった時に、この記事を見ればまた VSCode のカスタマイズが復元できる。

ここでは **Visual Studio Code の見た目に影響するものについてのみ** 取り上げる。

## スクリーンショット

![Editor](/images/ultimate-vscode-customization-2024/ss-editor.png)
*Editor*

![Tarminal](/images/ultimate-vscode-customization-2024/ss-terminal.png)
*Terminal*

![Zen Mode](/images/ultimate-vscode-customization-2024/ss-zen-mode.png)
*Zen Mode*

## 環境

### Visual Studio Code

- バージョン: `1.94.2 (user setup)`
- コミット: `384ff7382de624fb94dbaf6da11977bba1ecd427`
- 日付: `2024-10-09T16:08:44.566Z`
- Electron: `30.5.1`
- ElectronBuildId: `10262041`
- Chromium: `124.0.6367.243`
- Node.js: `20.16.0`
- V8: `12.4.254.20-electron.0`
- OS: `Windows_NT x64 10.0.22631`

## 基本方針

- シンプルかつミニマルな見た目にこだわる
- デフォルトで問題ないのであれば弄らない

## カスタマイズ

### 配色テーマ

配色テーマは "Ayu Mirage Bordered" を使用する。コントラストが丁度良くて、カラフルで見やすい。

```json
"workbench.colorTheme": "Ayu Mirage Bordered"
```

### ファイルアイコンテーマ

ファイルアイコンテーマは "Catppuccin Macchiato" を使用する。好みの見た目かつピクセルパーフェクト表示でいい感じ。

```json
"workbench.iconTheme": "catppuccin-macchiato"
```

### 製品アイコンのテーマ

製品アイコンのテーマは "JetBrains Idea Product Icon Theme" を使用する。あまりこだわりは無いけど、ミニマルで美しい見た目でいい感じ。

```json
"workbench.productIconTheme": "jetbrains-idea-product-icon-theme"
```

### エディタのフォント

エディタのフォントには `UDEV Gothic 35NFLG` を使用する。個人的にかなり見やすくてお気に入り。([UDEV Gothic - GitHub](https://github.com/yuru7/udev-gothic))

- "JetBrains Mono" と "BIZ UDゴシック" と "Nerd Fonts" を合成したフォント
- 全角3文字分の幅と半角5文字分の幅が同じ。
- 小文字アルファベットの高さが高めで視認性が高い
- リガチャに対応している

:::message
ターミナルのフォントは、デフォルトではエディタのフォントを継承する設定になっているため、この設定を追加した時点でエディタとターミナルの両方のフォントが変更されます。
:::

```json
"editor.fontFamily": "'UDEV Gothic 35NFLG'"
```

### 行の強調表示

デフォルトで行の強調表示は有効になっているが、以下の点を追加でカスタマイズする。

- 行番号が表示される部分も強調表示する
- エディタにフォーカスがある場合のみ強調表示を有効にする

```json
{
    "editor.renderLineHighlight": "all",
    "editor.renderLineHighlightOnlyWhenFocus": true
}
```

### カーソルの点滅

デフォルトの点滅よりもスムーズで見た目がいいので "phase" を指定。

```json
"editor.cursorBlinking": "phase"
```

### カーソルのキャレットアニメーション

カーソルが飛んだときに見失ってしまうことが多いため、キャレットの移動にアニメーションを設定する。

```json
"editor.cursorSmoothCaretAnimation": "explicit"
```


### エクスプローラーのインデント

個人的にデフォルトのインデントは浅くて見づらいので調整する。`24` を設定するとちょうどファイルアイコン一つ分 (とアイコンと文字の間のマージンの合計) のインデントになる。

```json
"workbench.tree.indent": 24
```

### レイアウトコントロールの非表示

VSCodeの画面右上あたりに表示されているサイドバーの表示とかを切り替えるメニューを非表示にする。
あまり使わない & 必要になればコマンドペインから操作できるので不要と判断。

```json
"workbench.layoutControl.enabled": false
```

### コマンドセンター非表示

VSCodeのタイトルバー中央に表示されているコマンドランチャーの表示を無効化する。いつもショートカットキーからアクセスしているため不要と判断。

```json
"window.commandCenter": false
```

### メニューバー非表示

VSCodeのタイトルバー左側に表示されているメニューを非表示にする。"toggle" に設定することで `Alt` キーで引き続きメニューにアクセス可能。

```json
"window.menuBarVisibility": "toggle"
```

### ウィンドウタイトル非表示

作業中にウィンドウタイトルを見ることが無いので不要と判断。
ウィンドウタイトルに半角スペース文字を設定することで消えたように見せている。

```json
"window.title": " "
```

## 参考にした記事

- [Minimal VScode UI! - DEV Community](https://dev.to/andrewgeorge/minimal-vscode-ui-343e)