# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Zenn (zenn.dev) のコンテンツ管理リポジトリ。zenn-cli を使って記事・本を管理し、GitHub連携でZennに公開される。

## Commands

```bash
# プレビューサーバー起動
pnpm exec zenn preview

# 新しい記事を作成
pnpm exec zenn new:article

# 新しい本を作成
pnpm exec zenn new:book

# スラッグ指定で記事作成
pnpm exec zenn new:article --slug my-article-slug
```

## Structure

- `articles/` — 記事のMarkdownファイル（YAMLフロントマター付き）
- `books/` — 本のディレクトリ（現在未使用）
- `images/` — 記事で使う画像。`images/<article-slug>/` のようにスラッグ名のディレクトリに格納

## Article Frontmatter Format

```yaml
---
title: "記事タイトル"
emoji: "🎉"
type: "tech"           # tech: 技術記事 / idea: アイデア
topics: ["topic1"]     # タグ（最大5つ）
publication_name: ""   # Publication名（任意）
published: true        # true で公開
---
```

## Image References

記事内の画像パスは `/images/<slug>/filename.png` の形式で参照する。
