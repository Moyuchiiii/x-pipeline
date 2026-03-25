---
name: x-run
description: moyuchi の X（Twitter）投稿を自動生成するスキル。note記事ドラフトからX投稿を生成するモード（from-article）と、Claude/Anthropic関連トレンドをスキャンして単独ツイートを生成するモード（standalone）がある。
---

# /x-run

moyuchi の X 投稿を生成する全自動パイプライン。

## 使い方

```
/x-run                              # モード選択画面
/x-run from-article {ドラフトパス}   # note記事 → X投稿変換
/x-run standalone                   # 単独ツイート / スレッド生成
/x-run trend                        # トレンドスキャン → ツイートアイデア生成
```

---

## Phase 0: コンテキスト読み込み

必ず最初に以下を読み込む:

- `context/profile.md` — X アカウント設計・ポジショニング
- `context/strategy.md` — ツイートタイプ・運用戦略
- `context/voice-samples.md` — X 用文体サンプル・NG パターン
- `context/performance.md` — 過去の成功パターン（あれば）
- note-pipeline の `context/performance.md` — note 記事の反応データ（参考）

---

## モード A: `from-article` — note記事 → X投稿変換

### 使用場面

- note 記事を公開するタイミング
- 記事の内容を X で拡散したいとき

### Step 1: 記事ドラフト読み込み

指定された `.md` または `.html` ファイルを読み込む。

読み取る情報:
- 記事タイプ（速報/実録/ノウハウ）
- 記事の核心（一番伝えたいこと）
- Before/After や数字（あれば）
- note URL（ドラフトの frontmatter から取得、なければ仮置き）

### Step 2: ツイートパターン判定

記事タイプに応じて出力パターンを決める:

| 記事タイプ | X 出力パターン |
|---|---|
| 速報（sokuho） | 単発ツイート1〜2本（速報 + note 誘導） |
| 実録（jituroku） | note 誘導ツイート + ノウハウスレッド（任意） |
| ノウハウ（knowhow） | ノウハウスレッド（5〜8ツイート） + note 誘導 |

### Step 3: ツイート生成

**単発ツイート（速報・note誘導）:**

```
[フック1行 — 事実ファースト or 一番驚いた点]
[自分の視点・感想（1〜2行）]
[詳しくは note に書いた → {URL}]
```

**ノウハウスレッド（5〜8ツイート）:**

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

---

## モード B: `standalone` — 単独ツイート / スレッド生成

### 使用場面

- note 記事とは別に、日常の実況・Tips を投稿したいとき
- ユーザーがトピックを指定して投稿内容を生成したいとき

### Step 1: トピック確認

ユーザーが指定したトピックを受け取る。
指定がなければ聞く:
- 実況ツイート（今やってること）
- Tips・ノウハウ（1つの技術・工夫）
- 実績報告（スキ数・案件・PV）

### Step 2: ツイート生成

`context/voice-samples.md` のサンプルを参考に生成。
単発 or スレッド（3ツイート以上の Tips の場合）を判断して出力。

---

## モード C: `trend` — トレンドスキャン → ツイートアイデア

### 使用場面

- 今日何をツイートするか決めたいとき
- Claude / Anthropic 関連の話題を拾いたいとき

### Step 1: トレンドスキャン

WebSearch で以下を検索（過去 24〜48 時間）:

```
"Claude Code" OR "Anthropic" OR "Claude" site:x.com OR site:twitter.com
"Claude" new feature release announcement
"Claude Code" tips OR tutorial
```

### Step 2: ツイートアイデア生成

スキャン結果から 3〜5 件のツイートアイデアを生成して提示。

各アイデアに:
- ツイートタイプ（速報/実況/ノウハウ/引用 RT）
- ドラフト文（完成形）
- 推奨投稿時間

ユーザーが選んだものを最終ドラフトとして保存。

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
created: YYYY-MM-DD
status: draft
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

---

## 品質ゲート

送信前に必ず確認:

- [ ] 1ツイート 280 字以内（改行込み）
- [ ] フックが1行目にある
- [ ] AI 感のある言い回しがない（「〜と言えるでしょう」等）
- [ ] ハッシュタグ 2 個以内
- [ ] 宣伝感がない（体験・実況ベース）
- [ ] note リンクが入るべき場所に入っている

**IMPORTANT: ツイートの送信はユーザーが行う。このスキルはドラフトを生成するだけ。**

---

## 関連スキル

- `content-engine` — 素材からプラットフォーム別コンテンツを生成
- `crosspost` — X + LinkedIn + Threads + Bluesky への同時展開
