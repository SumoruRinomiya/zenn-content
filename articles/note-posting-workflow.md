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

実際のダッシュボードではこのように表示されます。アイキャッチ画像・ファイル選択・HTMLコピーボタンがひとつの画面に収まっています。

![ダッシュボード Note プレビュータブ（記事一覧）](/images/articles/note-posting-workflow/dashboard_note_tab.png)

別の記事ファイルを選択すると、アイキャッチとプレビューが切り替わります。

![ダッシュボード Note プレビュータブ（記事選択後）](/images/articles/note-posting-workflow/dashboard_note_02.png)

---

## アイキャッチ画像の生成（DALL-E 3）

### 方針：キャラクターの外見をプロンプトに固定する

DALL-E 3 はシーン・構図の再現品質が高く、テキストプロンプトを忠実に反映します。参照画像は使えないため、キャラクターの外見説明をプロンプトに毎回固定で埋め込む方針にしました。

### 実装（gen_eyecatch_gpt.mjs）

出力先とシーン説明だけ渡せばアイキャッチを生成できるスクリプトを用意しました。

```javascript
// gen_eyecatch_gpt.mjs（抜粋）
const characterDescription = `A small, cute anime-style AI assistant girl named Sumoru.
Appearance: short white/silver bob hair, blue eyes, white sleeveless uniform
with a small collar and a glowing blue AI badge on the chest, petite figure.
Color theme: white, light pink, light blue.`;

const fullPrompt = `${characterDescription}

Scene: ${scenePrompt}

Style: Wide cinematic 16:9 anime illustration, high quality,
soft neon lighting with purple and pink tones, dark cosmic background.`;

const res = await fetch("https://api.openai.com/v1/images/generations", {
  method: "POST",
  headers: { "Authorization": `Bearer ${apiKey}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "dall-e-3",
    prompt: fullPrompt,
    size: "1792x1024",
    response_format: "b64_json",
  }),
});
```

使い方は引数でシーンを渡すだけです。

```bash
node gen_eyecatch_gpt.mjs /tmp/eyecatch.png \
  "Sumoru is sitting at a glowing desk looking at two screens — left shows raw Markdown, right shows a beautifully formatted blog article."
```

毎回外見説明が固定されるため、生成ごとのばらつきを最小限に抑えられます。

---

## まとめ

| 課題 | 解決策 |
|---|---|
| Markdown の記法が文字として出る | marked.js で HTML 変換 + ClipboardAPI で `text/html` コピー |
| 貼り付けると全文太字になる | DOMParser でインラインスタイルを要素ごとに付与 |
| Note 記事のプレビュー環境がない | Next.js ダッシュボードに Note タブを追加 |
| 記事画像の表示 | `/api/notes-images/[filename]` エンドポイントで配信 |
| アイキャッチ生成 | DALL-E 3 + キャラクター外見をプロンプトに固定して一貫性を確保 |

**注意**：Note.com の利用規約では自動投稿ツールの使用が制限されています。この実装はあくまで「コピー補助」であり、実際の投稿操作はブラウザ上で手動で行っています。

体験記（スモル視点）は Note に書いています → **[Zennの記事リンク]**
