---
name: x-run
description: moyuchi の X（Twitter）投稿を自動生成するスキル。Notionの note記事DB を監視して「Xステータス:未投稿」の記事からツイートを生成する。note記事 → X変換（from-article）、単独ツイート生成（standalone）、トレンドスキャン（trend）の3モード。
allowed-tools: WebSearch, WebFetch, Write, Read, Glob, Grep, mcp__notion__notion-search, mcp__notion__notion-fetch, mcp__notion__notion-update-page, mcp__notion__notion-create-pages
---

# /x-run

moyuchi の X 投稿を生成する全自動パイプライン。

## 使い方

```
/x-run                              # Notionの未投稿記事一覧を表示して選択
/x-run from-article {ドラフトパス}   # note記事ドラフトを直接指定して変換
/x-run standalone                   # 単独ツイート / スレッド生成
/x-run trend                        # トレンドスキャン → ツイートアイデア + note記事依頼
```

---

## Phase 0: コンテキスト読み込み

必ず最初に以下を読み込む:

- `context/profile.md` — X アカウント設計・ポジショニング
- `context/strategy.md` — ツイートタイプ・運用戦略
- `context/voice-samples.md` — X 用文体サンプル・NG パターン
- `context/performance.md` — 過去の成功パターン（あれば）

### Notion から「未投稿」記事を取得

Notion「note記事管理」DB（data_source: `collection://812aa728-8d3e-42e4-a9cd-6a91c303b2c2`）を参照し、`Xステータス: 未投稿` のレコードを取得する。

引数なしで `/x-run` を実行した場合は、未投稿記事の一覧を提示してユーザーに選ばせる:

```
📋 X未投稿の note 記事:
1. [記事タイトル] - 公開日: YYYY-MM-DD（速報/実録/ノウハウ）
2. [記事タイトル] - 公開日: YYYY-MM-DD
...

どの記事のツイートを生成しますか？（番号 or "all" で全件）
```

---

## モード A: `from-article` — note記事 → X投稿変換

### Step 1: 記事読み込み

引数がファイルパスなら直接読み込む。
引数なし（Phase 0 でユーザーが選択した場合）は Notion レコードから `note URL` と `草稿パス` を取得してファイルを読む。

読み取る情報:
- 記事タイプ（速報/実録/ノウハウ）
- 記事の核心（一番伝えたいこと）
- Before/After や数字（あれば）
- note URL（frontmatter or Notion レコードから取得）

### Step 2: ツイートパターン判定

| 記事タイプ | X 出力パターン |
|---|---|
| 速報（sokuho） | 単発ツイート 1〜2 本（速報 + note 誘導） |
| 実録（jituroku） | note 誘導ツイート + ノウハウスレッド（任意） |
| ノウハウ（knowhow） | ノウハウスレッド（5〜8 ツイート） + note 誘導 |

### Step 3: ツイート生成

**単発ツイート（速報・note誘導）:**

```
[フック1行 — 事実ファースト or 一番驚いた点]
[自分の視点・感想（1〜2行）]
[詳しくは note に書いた → {URL}]
```

**ノウハウスレッド（5〜8 ツイート）:**

```
ツイート1（フック）:
  [「これ知らない人多すぎる」「〇〇したら〇〇になった」系のフック]

ツイート2〜6（中身）:
  [1ツイート1アイデア。箇条書き禁止。文章で書く]

ツイート7（まとめ + 誘導）:
  [まとめ1行 + 「詳しくは note に書いた」 + URL]
```

**文体ルール:**
- `context/voice-samples.md` のサンプルに合わせる
- 一人称: 「自分」「僕」
- 1ツイートは最大 280 字（改行込み）
- ハッシュタグ: スレッドの最後のツイートのみ 0〜2 個
- AI 感のある言い回し禁止

### Step 4: Notion ステータス更新

草稿保存後、Notion の note記事 DB レコードを更新:
- `Xステータス`: 未投稿 → **草稿完成**
- `X草稿パス`: 保存したドラフトファイルのパス

---

## モード B: `standalone` — 単独ツイート / スレッド生成

### Step 1: トピック確認

ユーザーが指定したトピックを受け取る。指定がなければ聞く:
- 実況ツイート（今やってること）
- Tips・ノウハウ（1つの技術・工夫）
- 実績報告（スキ数・案件・PV）

### Step 2: ツイート生成

`context/voice-samples.md` のサンプルを参考に生成。単発 or スレッドを判断して出力。

---

## モード C: `trend` — トレンドスキャン → ツイート + note依頼

### Step 1: トレンドスキャン

WebSearch で以下を検索（過去 24〜48 時間）:

```
"Claude Code" OR "Anthropic" OR "Claude" new release announcement
"Claude Code" tips OR tutorial OR workflow
```

### Step 2: ツイートアイデア生成

スキャン結果から 3〜5 件のツイートアイデアを生成して提示。各アイデアに:
- ツイートタイプ（速報/実況/ノウハウ/引用 RT）
- ドラフト文（完成形）
- 推奨投稿時間

### Step 3: note 記事依頼（→ Notion ネタ帳）

ツイートアイデアのうち「深掘りすれば note 記事になる」ものを選別し、ユーザーに確認する:

```
📝 note 記事にできそうなネタ:
- [トピック名]: [一言説明]（推定スコア: ○点）

Notionのネタ帳に追加しますか？
```

承認されたら Notion「ネタ帳」DB（data_source: `collection://1a603b4f-d1e4-4ed7-8c75-c7c0a5b7e595`）に追加:
- タイトル: ネタの仮タイトル
- ソース: **X発案**
- ステータス: 未使用
- メモ: X でのトレンド情報、スキャン日時

---

## Phase 出力: 保存

### 保存先

- スレッド: `drafts/threads/draft_{日付}_{トピック要約}.md`
- 単発: `drafts/singles/draft_{日付}_{トピック要約}.md`

### ファイルフォーマット

```markdown
---
type: thread | single
source_article: {note記事パス or "standalone"}
source_notion_id: {Notion レコード ID or ""}
created: YYYY-MM-DD
status: draft
note_url: {URL or TBD}
---

# ツイート草稿

## ツイート1
{本文}

## ツイート2
{本文}
...

## 投稿メモ
- 推奨投稿時間: {時間帯}
- ハッシュタグ案: {0〜2個}
- note URL: {URL or TBD}
```

### コンソール出力

- 生成したツイート数
- ファイルパス
- 推奨投稿時間
- Notion 更新状況

---

## 投稿完了後の処理

ユーザーが X に投稿したら、Notion レコードを最終更新:
- `Xステータス`: 草稿完成 → **投稿済み**
- `XURL`: 投稿した X のポスト URL

実行コマンド例:
```
/x-run done {notion_record_id} {x_post_url}
```

---

## 品質ゲート

- [ ] 1ツイート 280 字以内（改行込み）
- [ ] フックが1行目にある
- [ ] AI 感のある言い回しがない
- [ ] ハッシュタグ 2 個以内
- [ ] 宣伝感がない（体験・実況ベース）
- [ ] note リンクが入るべき場所に入っている

**IMPORTANT: ツイートの送信はユーザーが行う。このスキルはドラフトを生成するだけ。**

---

## 関連スキル

- `content-engine` — 素材からプラットフォーム別コンテンツを生成
- `crosspost` — X + LinkedIn + Threads + Bluesky への同時展開
