# Design Document
## OpenBot + AG-UI + CrewAI + Local vLLM Demo

## 1. 目的

OpenBot、AG-UI、CrewAI、ローカル vLLM 上の Qwen を組み合わせて、
「複数の AI Agent が協調し、人間が途中で確認・承認できる」最小デモを構築する。

最初のユースケースは **AI 提案書レビュー・チーム** とする。

ユーザーが Markdown の提案書を渡すと、複数の Agent が順番に分析・レビュー・修正し、
OpenBot 上で進行状況と結果を確認できる。

本デモでは以下を重点的に示す。

1. CrewAI による Multi-Agent orchestration
2. AG-UI による Agent と UI 間のイベント連携
3. OpenBot による Agent workspace / Human-in-the-loop
4. vLLM + Qwen によるローカル LLM 実行
5. LLM Runtime と Agent Runtime の分離

---

## 2. デモの主張

このデモは単なるチャットボットではなく、以下の構造を持つ。

```text
User
  |
  v
OpenBot
  |
  | AG-UI
  v
CrewAI Flow
  |
  v
Analyst Agent
  |
  v
Risk Reviewer Agent ----> Human Approval (critical risk 時のみ)
  |                            |
  v                            v
Editor Agent  <----------------+
  |
  v
Local vLLM API
  |
  v
Qwen2 / Qwen2.5
```

役割は以下。

| Component | Responsibility |
|---|---|
| OpenBot | ユーザー操作、結果表示、ファイル入力、承認 UI |
| AG-UI | Agent 実行状態、streaming、tool call、HITL のイベント連携 |
| CrewAI | Multi-Agent orchestration、役割分担、実行順序 |
| vLLM | OpenAI compatible API を提供する LLM runtime |
| Qwen | 推論、文章解析、レビュー、改善案生成 |

---

## 3. 対象ユースケース

### AI 提案書レビュー・チーム

ユーザーが `proposal.md` を指定し、次のように依頼する。

```text
この提案書をレビューしてください。
技術、リスク、経営観点から問題点を整理し、改善版を作ってください。
```

CrewAI は以下の Agent を起動する。

### 3.1 Analyst Agent

役割:
- 提案書の目的を整理する
- 対象ユーザーを整理する
- 課題を抽出する
- KPI / 成果指標を抽出する
- 前提を整理する
- 曖昧な要件を指摘する

出力例:

```markdown
## Proposal Analysis

### Goal
問い合わせ業務の効率化

### Target
カスタマーサポート担当者

### KPI
- 平均対応時間 -50%
- FAQ 自動回答率 70%
```

### 3.2 Risk Reviewer Agent

役割:
- 技術リスク
- セキュリティリスク
- 個人情報 / データ流出リスク
- コスト
- 運用リスク
- AI の不確実性
- Human-in-the-loop の必要箇所

重大なリスクがある場合、Human-in-the-loop を要求する。

例:

```text
顧客データが外部 LLM に送信される可能性があります。

[Continue]
[Modify]
```

### 3.3 Editor Agent

役割:
- Analyst と Reviewer の出力を統合する
- 不明確な部分を修正する
- 改善された提案書を生成する

最終出力:

```text
proposal_reviewed.md
```

---

## 4. MVP スコープ

最初のバージョンでは以下のみ実装する。

### 実装する

- OpenBot
- AG-UI
- CrewAI
- vLLM
- Qwen2 / Qwen2.5
- Markdown ファイル入力
- 3 Agent
- Agent の進行状況表示
- 1 回の Human-in-the-loop
- 最終 Markdown 出力

### 実装しない

初期デモでは以下は入れない。

- RAG / Vector DB / Knowledge Graph
- Kafka / RabbitMQ
- Kubernetes
- MCP
- OAuth / 認証 / RBAC
- Web Search / 外部 SaaS
- Agent Marketplace
- Langfuse / OpenTelemetry

正式な一覧は [`spec_todo.md`](./spec_todo.md) NFR-003 を参照。

理由は、最初に

```text
Qwen -> CrewAI -> AG-UI -> OpenBot
```

の通信経路を確実に動作させるため。

---

## 5. 技術アーキテクチャ

```text
+-----------------------------------------------------+
|                     OpenBot                         |
|                                                     |
| Prompt input                                        |
| File upload                                         |
| Agent progress                                      |
| Human approval                                      |
| Final result                                        |
+-----------------------------+-----------------------+
                              |
                              | AG-UI
                              |
+-----------------------------v-----------------------+
|                  CrewAI Application                 |
|                                                     |
| CrewAI Flow                                         |
|                                                     |
|  +------------+   +--------------+   +-----------+ |
|  | Analyst    |-->| Risk Reviewer|-->| Editor    | |
|  +------------+   +--------------+   +-----------+ |
|                         |                           |
|                         v                           |
|                    HITL Event                       |
+-----------------------------+-----------------------+
                              |
                              | OpenAI Compatible API
                              |
+-----------------------------v-----------------------+
|                         vLLM                        |
|                                                     |
| POST /v1/chat/completions                           |
|                                                     |
+-----------------------------+-----------------------+
                              |
                              v
                    Qwen2 / Qwen2.5
```

---

## 6. LLM Runtime

### 推奨モデル

最初は以下を推奨する。

```text
Qwen/Qwen2.5-7B-Instruct
```

Tool Calling を利用する場合は Qwen2 より Qwen2.5 を優先する。

### vLLM 起動例

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

Tool Calling を利用しない場合は簡略化できる。

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --host 0.0.0.0 \
  --port 8000
```

### API

```text
Base URL:
http://localhost:8000/v1
```

主要 endpoint:

```text
POST /v1/chat/completions
```

---

## 7. CrewAI LLM 設定

概念例:

```python
from crewai import LLM

llm = LLM(
    model="openai/Qwen/Qwen2.5-7B-Instruct",
    base_url="http://localhost:8000/v1",
    api_key="EMPTY",
    temperature=0.2,
)
```

全 Agent は初期段階では同じ LLM を共有する。

```text
Analyst
   |
Risk Reviewer
   |
Editor
   |
Qwen2.5
```

将来的には Agent ごとにモデルを変えられる。

---

## 8. CrewAI 設計

### Flow

```text
START
  |
  v
Load Proposal
  |
  v
Analyst
  |
  v
Risk Reviewer
  |
  +---- no critical risk ----+
  |                          |
  |                          v
  |                       Editor
  |
  +---- critical risk ----+
                             |
                             v
                       Human Approval
                             |
                +------------+------------+
                |                         |
                v                         v
             Continue                 Modify
                |                         |
                +------------+------------+
                             |
                             v
                           Editor
                             |
                             v
                            END
```

### 状態

Flow 内では以下の状態を保持する。

```python
class ReviewState:
    proposal_text: str
    analysis_result: str
    risk_result: str
    approval_required: bool
    approval_status: str | None
    final_document: str | None
```

---

## 9. AG-UI イベント設計

UI には最終結果だけでなく Agent の途中状態を流す。

想定イベント:

```text
workflow_started
agent_started
agent_message
agent_completed
approval_required
approval_received
workflow_completed
workflow_failed
```

例:

```json
{
  "type": "agent_started",
  "agent": "risk_reviewer"
}
```

例:

```json
{
  "type": "approval_required",
  "reason": "External LLM data transmission risk",
  "options": [
    "continue",
    "modify"
  ]
}
```

---

## 10. OpenBot UI

UI のビジュアルデザインは、リポジトリ直下の [`DESIGN.md`](../DESIGN.md) を正とする。
形式は Google が提唱する [DESIGN.md](https://github.com/google-labs-code/design.md)
（YAML front matter のデザイントークン + Markdown の設計意図）に従う。

- 色・タイポグラフィ・余白・角丸・コンポーネントは `DESIGN.md` のトークンを使い、コードに直書きしない
- デザイン変更は **`DESIGN.md` を先に更新** してから UI に反映する
- 変更時は `npx @google/design.md lint DESIGN.md` でエラー・警告がないことを確認する
- OpenBot のテーマ設定、または独自 UI を作る場合のスタイルの元データとして使う
  （必要に応じて `export --format css-tailwind` 等で変換する）

最低限以下を表示する。

### 入力

- Prompt
- Markdown file

### Workflow 表示

```text
Proposal Review

Analyst           Done
Risk Reviewer     Running
Editor            Waiting
```

### Approval

```text
Security Risk Detected

顧客データが外部 LLM に送信される可能性があります。

[Continue]
[Modify]
```

### 結果

```text
proposal_reviewed.md
```

---

## 11. ディレクトリ構成案

```text
ai_agent/
|
+-- CLAUDE.md
+-- README.md
+-- DESIGN.md
+-- docs/
|   +-- design_doc_bot.md
|   +-- spec_todo.md
|   +-- agent_contracts.md
|   +-- development_workflow.md
|   +-- multi_bot_architecture.md
|
+-- app/
|   +-- main.py
|   +-- config.py
|
+-- agents/
|   +-- analyst.py
|   +-- risk_reviewer.py
|   +-- editor.py
|
+-- flows/
|   +-- proposal_review.py
|
+-- llm/
|   +-- client.py
|
+-- agui/
|   +-- server.py
|   +-- events.py
|
+-- prompts/
|   +-- analyst.md
|   +-- risk_reviewer.md
|   +-- editor.md
|
+-- examples/
|   +-- proposal.md
|
+-- outputs/
|   +-- .gitkeep
|
+-- tests/
    +-- test_llm.py
    +-- test_agents.py
    +-- test_flow.py
```

---

## 12. 実装フェーズ

実装フェーズは [`spec_todo.md`](./spec_todo.md) の「10. Implementation TODO」を正とする。
各 Phase の TODO と Done Condition は spec_todo.md を参照。

| Phase | 名前 | ゴール |
|---|---|---|
| 0 | Project Bootstrap | Python 環境・`pyproject.toml`・`.env.example`・sample proposal を用意 |
| 1 | vLLM | `Python -> vLLM -> Qwen` で応答を取得 |
| 2 | CrewAI Single Agent | Analyst Agent が local Qwen で動く |
| 3 | Three Agents | `proposal -> Analyst -> Risk Reviewer -> Editor` が CLI 上で動く |
| 4 | CrewAI Flow | `ReviewState` を持った workflow として完走する |
| 5 | AG-UI | 外部 UI から Agent の進捗を観測できる |
| 6 | OpenBot | OpenBot から workflow を実行できる |
| 7 | Human-in-the-loop | Risk Reviewer が workflow を pause し、Continue / Modify で再開する |
| 8 | Demo Polish | demo input・出力レイアウト・5 分デモ手順を整える |

---

## 13. 非機能要件

### ローカル実行

LLM 推論はローカルで完結できること。

### 再現性

以下を固定する。

```text
Python version
CrewAI version
AG-UI version
vLLM version
Qwen model
```

### Observability

MVP では Python logging のみ。

```text
timestamp
workflow_id
agent
event
latency
error
```

将来は OpenTelemetry / Langfuse を追加する。

### セキュリティ

MVP は localhost 前提。

```text
0.0.0.0
```

で公開する場合はネットワーク制御を行う。

---

## 14. 成功条件

以下が動けば MVP 完了。

1. OpenBot から Markdown を渡せる
2. CrewAI workflow が開始される
3. Analyst が分析する
4. Risk Reviewer がレビューする
5. UI に Agent 状態が表示される
6. Risk がある場合に workflow が停止する
7. ユーザーが承認できる
8. Editor が最終 Markdown を作る
9. Qwen は local vLLM から利用される
10. 外部 LLM API がなくてもデモできる

---

## 15. 将来拡張

MVP 完成後、以下に発展させる。

```text
OpenBot
   |
AG-UI
   |
Agent Router
   |
Agent Registry
   |
+----------+----------+----------+
|          |          |          |
Research   Cost     Security   Domain
Agent      Agent     Agent      Agent
|
Event Stream
|
Kafka / RabbitMQ
|
Policy / Guardrail
|
Observability
```

さらに以下を追加できる。

- Agent Registry
- Agent Marketplace
- Agent metadata
- Agent rating
- Guardrail
- RBAC / ABAC
- MCP Gateway
- Knowledge Graph
- RAG
- Kafka / RabbitMQ
- Langfuse
- OpenTelemetry
- Kubernetes

最終的には Agentic Mesh の最小 PoC として発展させる。
