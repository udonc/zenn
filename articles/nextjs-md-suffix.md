---
title: "[Next.js] URL末尾に .md つけたらMarkdownをそのまま返す！"
published: false
type: "tech"
topics: ["nextjs", "markdown"]
emoji: "🪄"
publication_name: "chot"
---

Next.jsのドキュメントやQiitaなどでは、URLの末尾に `.md` を付けるとページ内容を**生のMarkdown**で取得できます。AIエージェントにコンテキストを渡したり、別クライアントから取り込んだりするのに便利なパターンです。

この記事では **Rewrites + Route Handlers（App Router）** で、`/post/hello` はHTML、`/post/hello.md` は `text/markdown` を返す“二刀流配信”を実装します。

## 前提

- Next.js 16.0.0 (App Router)
- React 19.2.0
- Markdownでコンテンツを管理している

## 動作イメージ

- `/post/hello-world` -> 通常のWebページ (HTML)
- `/post/hello-world.md` -> Markdownがレスポンスされる

### Demo

https://next-md-blog-template-black.vercel.app

### GitHub

https://github.com/udonc/next-md-blog-template

## ディレクトリ構成

シンプルな個人ブログサイトを想定しています。
`content/` ディレクトリに markdown を作成してコンテンツを追加していくイメージです。

```plain
.
├── app/
│   ├── post/
│   │   └── [slug]/
│   │       ├── md/
│   │       │   └── route.ts # markdown を返すエンドポイント
│   │       ├── layout.tsx
│   │       └── page.tsx # 記事詳細ページ
│   ├── layout.tsx
│   └── page.tsx
├── content/ # ここに記事を追加していくイメージ
│   └── hello-world.md
└── next.config.ts
```

## HTMLページ側

slugからmarkdownファイルをインポートしてHTMLとしてレンダリングしているだけです。Next.js は標準でMDXをサポートしているので、ドキュメントを参考に実装していきます。

[Guides: MDX | Next.js](https://nextjs.org/docs/app/guides/mdx)

```tsx:app/post/[slug]/page.tsx
import type { Metadata } from "next";

type Props = {
  params: Promise<{ slug: string }>;
};

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const { frontmatter } = (await import(`@/content/${slug}.md`)) as {
    default: React.ComponentType;
    frontmatter: { title?: string; description?: string };
  };

  return {
    title: frontmatter.title ?? "Untitled Post",
    description: frontmatter.description,
  };
}

export default async function Page({ params }: Props) {
  const { slug } = await params;
  const { default: Post } = (await import(`@/content/${slug}.md`)) as {
    default: React.ComponentType;
    frontmatter: { title?: string; description?: string };
  };

  return <Post />;
}
```

## Markdownを返す `route.ts`

Markdownテキストを返すエンドポイントをRoute Handlerとして実装します。
`/post/hello-world/md` にアクセスすると `hello-world.md`の内容が `text/markdown` として返ってくるようになります。

```ts:app/post/[slug]/md/route.ts
import fs from "node:fs/promises";
import path from "node:path";

/**
 * マークダウンファイルのパスを取得する関数
 */
async function getMarkdownFilePath(slug: string) {
  const filename = `${slug}.md`;
  const filepath = path.join(process.cwd(), "content", filename);

  return filepath;
}

export async function GET(
  _request: Request,
  { params }: { params: Promise<{ slug: string }> },
) {
  const { slug } = await params;

  const filepath = await getMarkdownFilePath(slug);

  // ファイルの存在確認
  const fileExists = await fs
    .access(filepath)
    .then(() => true)
    .catch(() => false);

  if (!fileExists) {
    return new Response("Not Found", { status: 404 });
  }

  // ファイルの内容を読み込む
  const content = await fs.readFile(filepath, "utf-8");

  return new Response(content, {
    headers: {
      "content-type": "text/markdown; charset=utf-8",
    },
  });
}
```

## Rewritesを使用してURLをハンドリングする

`next.config.ts` で `rewrites` を使用して `/post/:slug.md` に来たリクエストを内部で `/post/:slug/md` に回します。

これにより `/post/hello-world.md` にアクセスして `/post/hello-world/md` と同じ内容がレスポンスされます。

```ts:next.config.ts
import createMdx from "@next/mdx";
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  pageExtensions: ["ts", "tsx", "md", "mdx"],
  rewrites: async () => {
    return [
      {
		// `/post/:slug.md` に来たアクセスを `/post/:slug/md` に回す
        source: "/post/:slug.md",
        destination: "/post/:slug/md",
      },
    ];
  },
};

const withMdx = createMdx({
  extension: /\.mdx?$/,
  options: {
    remarkPlugins: ["remark-frontmatter", "remark-mdx-frontmatter"],
  },
});

export default withMdx(nextConfig);
```

## おわりに

二刀流配信によって、人間には読みやすく、AIには扱いやすい――それぞれに最適なフォーマットで記事を届けられるようになりました。
これからは **AX（Agent Experience）**、つまり“エージェントにとっての体験” も意識した記事配信が求められる時代になりそうです。
