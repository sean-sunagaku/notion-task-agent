---
name: notion-task-agent
description: |
  Notion DB と連携して毎日のタスクを AI が提案するスキル。
  「今日何やる？」「タスク提案して」「today's tasks」と聞かれたらこのスキルを使う。
  Backlog から最大6つのタスクを選び、期限・優先度・定期タスクを考慮して提案する。
  提案後、ユーザーの承認を得て Notion のステータスを Today に更新する。
  Triggers: "今日何やる", "タスク提案", "今日のタスク", "what should I do today",
  "today's tasks", "タスク確認", "何やるべき", "today todo"
---

# Notion Task Agent

Notion DB からタスクを読み取り、今日やるべき6つのタスクを提案する。

## 前提条件

- Notion MCP が接続済みであること（`mcp__claude_ai_Notion__*` または `mcp__notion__*` ツール）
- タスク管理用の Notion DB が存在すること

## DB スキーマ

| プロパティ | 型 | 値 |
|-----------|-----|-----|
| タスク名 | title | - |
| ステータス | select | Backlog / Today / In Progress / Done |
| カテゴリ | select | アプリ開発 / 記事・発信 / 仕事 / 学習 |
| 期限 | date | ISO-8601 |
| 優先度 | select | High / Medium / Low |
| 種類 | select | 単発 / 定期 / 習慣 |

## タスク提案ロジック

毎朝の提案は以下のロジックで6つを選ぶ：

### Step 1: 既に Today / In Progress のタスクを確認

まず現在 Today または In Progress のタスクを取得する。
これらは既にユーザーが着手中なので、提案枠に含める。

### Step 2: 定期タスクを含める

種類が「定期」のタスクは毎日 Today に入る。
定期タスクが Backlog にある場合は自動的に Today 候補に含める。

### Step 3: 残り枠を Backlog から選ぶ

6つ - (Step 1 + Step 2 の件数) = 残り枠

残り枠は Backlog から以下の優先順位で選ぶ：
1. **期限が今日から3日以内** のタスク（緊急）
2. **期限が今日から7日以内** で **優先度 High** のタスク
3. **優先度 High** で期限なしのタスク
4. **優先度 Medium** のタスク
5. **優先度 Low** のタスク

同じ優先度の場合は、期限が近いものを優先する。
期限がないもの同士では、カテゴリのバランスを考慮する（同じカテゴリばかりにならないように）。

### Step 4: ユーザーに提案

以下のフォーマットで提案する：

```
今日やること（6件）:

1. [タスク名] - カテゴリ | 優先度 | 期限: YYYY-MM-DD
2. [タスク名] - カテゴリ | 優先度
3. ...

入れ替えたいタスクがあれば言ってください。
OKなら Notion を更新します。
```

### Step 5: 承認後に Notion 更新

ユーザーが OK したら、選んだタスクのステータスを Today に更新する。
`mcp__claude_ai_Notion__notion-update-page` を使って各タスクのステータスを変更する。

## Notion DB の読み取り方法

### 方法 A: Claude AI Notion MCP（推奨）

`mcp__claude_ai_Notion__notion-search` でタスク管理 DB を検索し、
`mcp__claude_ai_Notion__notion-fetch` でデータを取得する。

### 方法 B: Notion API MCP

`mcp__notion__API-post-search` で DB を検索し、
`mcp__notion__API-query-data-source` でクエリする。

フィルタ例（Backlog のタスクを取得）：
```json
{
  "filter": {
    "property": "ステータス",
    "select": { "equals": "Backlog" }
  },
  "sorts": [
    { "property": "期限", "direction": "ascending" },
    { "property": "優先度", "direction": "ascending" }
  ]
}
```

## 完了処理

ユーザーが「終わった」「完了」と言ったら、該当タスクのステータスを Done に更新する。

## 新規タスク追加

ユーザーが新しいタスクを言ったら、DB に追加する。
カテゴリと優先度はコンテキストから推測し、ユーザーに確認する。

## 振り返り（オプション）

1日の終わりに「振り返り」と言われたら：
- Today のまま Done にならなかったタスクを表示
- 明日に持ち越すか、Backlog に戻すか確認
- Done にしたタスクの数をカウントして報告
