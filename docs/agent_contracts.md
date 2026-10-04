# Agent Contracts

各 Agent の入出力契約、Human-in-the-loop、AG-UI イベントの定義。

> 詳細設計は [`design_doc_bot.md`](./design_doc_bot.md)、
> イベント payload は [`spec_todo.md`](./spec_todo.md) の「API / Event Contract」を参照。

---

## 1. Agent 入出力

| Agent | 入力 | 出力 |
|---|---|---|
| **Analyst** | proposal markdown | `goal` / `target` / `problem` / `KPI` / `assumptions` / `missing requirements` |
| **Risk Reviewer** | original proposal, analyst result | `risks` / `severity` / `mitigation` / `approval_required` |
| **Editor** | original proposal, analysis, risks, human decision | revised markdown |

実行順序:

```text
proposal ──▶ Analyst ──▶ Risk Reviewer ──▶ (HITL) ──▶ Editor ──▶ outputs/
```

---

## 2. Human-in-the-loop

1. Risk Reviewer は重大なリスクを検出した場合に `approval_required = true` を返す
2. CrewAI Flow はそこで **停止** し、AG-UI に `approval_required` event を送る
3. 人間の入力を受信後に再開する

| 入力 | 意味 |
|---|---|
| `continue` | そのまま Editor へ進む |
| `modify` | 人間の修正指示を Editor に渡して進む |

```text
Risk detected ─▶ workflow paused ─▶ human input ─▶ workflow resumed
```

---

## 3. AG-UI イベント

[`spec_todo.md`](./spec_todo.md) FR-006 に従い、以下の 8 event を UI に通知する。

| Event | タイミング | 実装 Phase |
|---|---|---|
| `workflow_started` | Flow 開始時 | 5 |
| `agent_started` | 各 Agent 開始時 | 5 |
| `agent_message` | Agent の途中出力 | 5 |
| `agent_completed` | 各 Agent 完了時 | 5 |
| `approval_required` | HITL で停止したとき | 7 |
| `approval_received` | 人間の入力を受信したとき | 7 |
| `workflow_completed` | 最終 Markdown 出力後 | 5 |
| `workflow_failed` | 例外発生時（握りつぶさない） | 5 |
