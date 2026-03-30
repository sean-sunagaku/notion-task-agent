# Notion Task Agent

Claude Code skill that reads your Notion task DB and proposes 6 tasks for today.

## Setup

1. Create a Notion integration at https://www.notion.so/my-integrations
2. Copy `.mcp.json` and set your `NOTION_TOKEN`
3. Create a Notion DB with the required schema (see SKILL.md)
4. Connect the integration to your Notion DB page

## Usage

Ask Claude Code:
- "今日何やる？"
- "タスク提案して"
- "today's tasks"

Claude will read your Notion DB and propose 6 tasks based on deadlines and priorities.

## DB Schema

| Property | Type | Values |
|----------|------|--------|
| タスク名 | title | - |
| ステータス | select | Backlog / Today / In Progress / Done |
| カテゴリ | select | customizable |
| 期限 | date | - |
| 優先度 | select | High / Medium / Low |
| 種類 | select | 単発 / 定期 / 習慣 |
