---
title: "AIアシスタントのNote投稿をMarkdown→HTMLコピーで半自動化した"
emoji: "📝"
type: "tech"
topics: ["nextjs", "claude", "typescript", "nodejs"]
published: false
---

![header](/images/articles/note-posting-workflow/header.png)

## はじめに

Note.com には公式の投稿 CLI がありません。Zenn には `zenn-cli` があり、Markdown ファイルをリポジトリで管理してプレビューしながら書けますが、Note はブラウザのエディタに直接入力する前提の設計になっています。

そのため、Markdown で書いた記事をそのまま Note エディタに貼り付けると `##`・`**` などの記法がそのまま文字として表示されてしまいます。

```
## 見出し  →  ## 見出し（記号がそのまま出る）
**太字**    →  **太字**（アスタリスクがそのまま出る）
```

この問題を解決するため、以下の方針で「Markdown → HTML クリップボードコピー」機能を自分のダッシュボードに実装しました。

---

## 方針

Note のリッチエディタは HTML をそのまま貼り付けると整形して表示します。つまり：

1. Markdown を HTML に変換する
2. `text/html` 型として ClipboardAPI でコピーする
3. Note エディタに Ctrl+V で貼り付ける

この流れで、Markdown の記法が正しく見出し・太字・リストとして反映されます。

---

## 実装：marked.js + DOMParser で Note 用 HTML を生成

### 基本変換（Markdown → HTML）

`marked` ライブラリで Markdown を HTML 文字列に変換します。

```bash
npm install marked
```

```typescript
import { marked } from "marked";

const html = await marked(markdownContent);
```

これだけで基本的な HTML は得られますが、Note に貼り付けると**全テキストが太字になる**問題が発生しました。

### 原因と対策：インラインスタイルの付与

Note のエディタがデフォルトで `font-weight: bold` を適用していたため、`<p>` タグなどに明示的に `font-weight: normal` を指定する必要がありました。

`DOMParser` を使ってHTMLをDOM操作し、各要素にインラインスタイルを付与します。

```typescript
function applyNoteStyles(html: string): string {
  const parser = new DOMParser();
  const doc = parser.parseFromString(html, "text/html");

  // 段落：ノーマルウェイト・行間確保
  doc.querySelectorAll("p").forEach((el) => {
    el.style.fontWeight = "normal";
    el.style.marginBottom = "1em";
    el.style.lineHeight = "1.8";
  });

  // 見出し
  doc.querySelectorAll("h1").forEach((el) => {
    el.style.fontSize = "1.8em";
    el.style.fontWeight = "bold";
    el.style.marginBottom = "0.5em";
  });
  doc.querySelectorAll("h2").forEach((el) => {
    el.style.fontSize = "1.4em";
    el.style.fontWeight = "bold";
    el.style.marginBottom = "0.5em";
  });

  // 太字（strong）は明示的に bold を保持
  doc.querySelectorAll("strong").forEach((el) => {
    el.style.fontWeight = "bold";
  });

  // リスト
  doc.querySelectorAll("li").forEach((el) => {
    el.style.fontWeight = "normal";
    el.style.lineHeight = "1.8";
  });

  return doc.body.innerHTML;
}
```

### HTML としてクリップボードにコピー

`text/plain` ではなく `text/html` として書き込むことで、貼り付け先が整形済み HTML として受け取ります。

```typescript
async function copyAsHtml(html: string) {
  const styledHtml = applyNoteStyles(html);
  const blob = new Blob([styledHtml], { type: "text/html" });
  const item = new ClipboardItem({ "text/html": blob });
  await navigator.clipboard.write([item]);
}
```

---

## 実装：Next.js ダッシュボードへのプレビュータブ追加

### ファイル自動検出

`/api/note-files` エンドポイントで `notes/blog_note_*.md` を glob して一覧を返します。

```typescript
// app/api/note-files/route.ts
import { glob } from "glob";
import path from "path";

export async function GET() {
  const files = await glob("notes/blog_note_*.md", {
    cwd: path.resolve(process.cwd(), ".."),
  });
  return Response.json({ files: files.sort() });
}
```

### 画像配信エンドポイント

Note 記事中の画像（`notes/` ディレクトリに配置）をブラウザから参照できるよう `/api/notes-images/[filename]` で配信します。

```typescript
// app/api/notes-images/[filename]/route.ts
import fs from "fs";
import path from "path";

export async function GET(
  _req: Request,
  { params }: { params: { filename: string } }
) {
  const filePath = path.resolve(
    process.cwd(),
    "../notes",
    params.filename
  );
  const buffer = fs.readFileSync(filePath);
  return new Response(buffer, {
    headers: { "Content-Type": "image/png" },
  });
}
```

### プレビュータブのコンポーネント

```tsx
// NoteTab.tsx（抜粋）
import ReactMarkdown from "react-markdown";

export function NoteTab({ content }: { content: string }) {
  const handleCopy = async () => {
    const html = await marked(content);
    await copyAsHtml(html);
    alert("HTMLとしてコピーしました！Note エディタに貼り付けてください。");
  };

  return (
    <div>
      <button onClick={handleCopy}>Note 用 HTML をコピー</button>
      <div className="preview">
        <ReactMarkdown>{content}</ReactMarkdown>
      </div>
    </div>
  );
}
```

---

## アイキャッチ画像の生成（DALL-E 3）

### キャラクター一貫性のためのプロンプト固定法

キャラクター画像を毎回同じ見た目で生成するには、外見を徹底的に言語化したプロンプトを固定して使い回す方法が現状最も安定しています。

```
A cute anime-style AI assistant character. Short black hair,
pale skin, wearing a white and purple dress. Gentle smile,
sitting at a desk with a laptop. Soft pastel background.
Digital illustration style.
```

### 参照画像 API の現状制限（2026年3月時点）

| API | 参照画像入力 | 備考 |
|---|---|---|
| OpenAI gpt-image-1 `/edits` | 対応（β） | 参照画像との一貫性は完全ではない |
| DALL-E 3 | 非対応 | プロンプトのみ |
| Gemini（実験的モデル） | 一部対応 | 品質・安定性に難あり |

結論として、プロンプトを詳細に固定して DALL-E 3 で生成する方法が、品質・コスト・手軽さのバランスが取れていました。

生成スクリプトの例：

```javascript
// gen_eyecatch.mjs
import OpenAI from "openai";
import fs from "fs";

const client = new OpenAI();

const response = await client.images.generate({
  model: "dall-e-3",
  prompt: `...(固定プロンプト)...`,
  n: 1,
  size: "1792x1024",
});

const url = response.data[0].url;
// url から画像をダウンロードして保存
```

---

## まとめ

| 課題 | 解決策 |
|---|---|
| Markdown の記法が文字として出る | marked.js で HTML 変換 + ClipboardAPI で `text/html` コピー |
| 貼り付けると全文太字になる | DOMParser でインラインスタイルを要素ごとに付与 |
| Note 記事のプレビュー環境がない | Next.js ダッシュボードに Note タブを追加 |
| 記事画像の表示 | `/api/notes-images/[filename]` エンドポイントで配信 |
| アイキャッチ生成 | DALL-E 3 + 固定プロンプトで一貫性を確保 |

**注意**：Note.com の利用規約では自動投稿ツールの使用が制限されています。この実装はあくまで「コピー補助」であり、実際の投稿操作はブラウザ上で手動で行っています。

体験記（スモル視点）は Note に書いています → **[Zennの記事リンク]**
