# Specification & TODO
## OpenBot + AG-UI + CrewAI + Local vLLM Demo

このドキュメントは以下を兼ねる。

- 要件定義
- 機能仕様
- 非機能仕様
- Acceptance Criteria
- 実装 TODO
- テスト TODO
- 将来拡張 backlog

[`design_doc_bot.md`](./design_doc_bot.md) を設計の基準とし、このファイルを実装進捗の基準とする。

---

# 1. Product Goal

ローカル LLM を利用した Multi-Agent workflow を OpenBot 上で実行し、
AI Agent の進行状況と Human-in-the-loop をユーザーに見せる。

最初のユースケース:

**AI 提案書レビュー・チーム**

---

# 2. User Story

ユーザーとして、

Markdown の提案書を AI に渡し、

- 提案内容を分析
- 技術 / セキュリティ / 運用リスクを確認
- 問題があれば途中で人間に確認
- 改善された提案書を生成

してほしい。

理由は、Multi-Agent + Human-in-the-loop の価値を
短時間のデモで理解できるようにするため。

---

# 3. System Context

```text
User
  |
OpenBot
  |
AG-UI
  |
CrewAI Flow
  |
+-------------------------+
| Analyst                 |
| Risk Reviewer           |
| Editor                  |
+-------------------------+
  |
vLLM
  |
Qwen2.5
```

---

# 4. Functional Requirements

## FR-001 Proposal Input

ユーザーは Markdown の提案書を入力できること。

入力方式:

- file path
- upload
- text

MVP では最低 1 方式が動けばよい。

### Acceptance

- [ ] Markdown を CrewAI workflow に渡せる
- [ ] 空ファイルを reject できる
- [ ] UTF-8 日本語を扱える

---

## FR-002 Analyst Agent

Analyst Agent は提案書から以下を抽出する。

- 目的
- 対象ユーザー
- 課題
- KPI
- 前提
- 不明確な要件

### Acceptance

- [ ] Analyst Agent が実行される
- [ ] 構造化された結果を返す
- [ ] 日本語 Markdown を処理できる

---

## FR-003 Risk Reviewer Agent

Risk Reviewer は以下を確認する。

- 技術リスク
- Security
- Privacy
- Cost
- Operation
- AI uncertainty
- Human intervention

出力:

```text
risk
severity
mitigation
approval_required
```

### Acceptance

- [ ] Reviewer が Analyst 出力を受け取る
- [ ] risk severity を返せる
- [ ] critical risk で approval_required=true になる

---

## FR-004 Editor Agent

Editor は以下を入力として最終提案書を生成する。

- original proposal
- analysis
- risk review
- human decision

### Acceptance

- [ ] 改善版 Markdown を生成する
- [ ] original proposal を完全に失わない
- [ ] review 内容を反映する

---

## FR-005 Multi-Agent Workflow

実行順序:

```text
Analyst
  |
Risk Reviewer
  |
Human Approval if required
  |
Editor
```

### Acceptance

- [ ] 上記順序で実行される
- [ ] Agent 間で context を引き継げる
- [ ] Agent failure を検出できる

---

## FR-006 AG-UI Event Streaming

以下の event を UI に通知する。

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

### Acceptance

- [ ] workflow start が UI に見える
- [ ] 現在動作中の Agent が分かる
- [ ] Agent 完了が分かる
- [ ] workflow 完了が分かる

---

## FR-007 Human-in-the-loop

重大リスク時に workflow を停止する。

UI:

```text
Risk Detected

[Continue]
[Modify]
```

### Acceptance

- [ ] approval_required event が送信される
- [ ] workflow が pause する
- [ ] human input を受け取る
- [ ] workflow が resume する

---

## FR-008 Final Output

最終結果を Markdown として保存する。

Default:

```text
outputs/proposal_reviewed.md
```

### Acceptance

- [ ] Markdown file が生成される
- [ ] UTF-8
- [ ] UI から結果を確認できる

---

# 5. LLM Requirements

## LLM-001 Local vLLM

LLM は local vLLM を利用できること。

Default:

```text
http://localhost:8000/v1
```

### Acceptance

- [ ] `/v1/chat/completions` に接続できる
- [ ] 外部 OpenAI API key なしで動く
- [ ] connection error を検知できる

---

## LLM-002 Model

Default:

```text
Qwen/Qwen2.5-7B-Instruct
```

Qwen2 でも基本デモは許容する。

Tool Calling を利用する場合は Qwen2.5 を推奨。

---

## LLM-003 Configuration

以下を environment variable から変更可能にする。

```text
VLLM_BASE_URL
VLLM_API_KEY
VLLM_MODEL
```

### Acceptance

- [ ] hard-code しない
- [ ] `.env.example` を用意する

---

# 6. Non-Functional Requirements

## NFR-001 Local First

MVP は外部クラウド LLM を必要としないこと。

- [ ] local environment only で demo 可能

---

## NFR-002 Logging

最低限以下を logging する。

```text
timestamp
workflow_id
agent
event
latency
error
```

- [ ] prompt 全文をログしない
- [ ] proposal 全文を default ではログしない

---

## NFR-003 Simplicity

MVP では以下を導入しない。

- [ ] RAG を導入しない
- [ ] Vector DB を導入しない
- [ ] Kafka を導入しない
- [ ] RabbitMQ を導入しない
- [ ] Kubernetes を導入しない
- [ ] Knowledge Graph を導入しない
- [ ] MCP を導入しない
- [ ] OAuth / 認証 / RBAC を導入しない
- [ ] Web Search / 外部 SaaS 連携を導入しない
- [ ] Agent Marketplace を導入しない
- [ ] Langfuse / OpenTelemetry を導入しない

---

# 7. Data Model

## ReviewState

最低限:

```python
proposal_text: str
analysis_result: str
risk_result: str
approval_required: bool
approval_status: str | None
final_document: str | None
```

- [ ] state model を定義
- [ ] validation を追加

---

# 8. API / Event Contract

## Agent Started

```json
{
  "type": "agent_started",
  "workflow_id": "string",
  "agent": "analyst"
}
```

## Agent Completed

```json
{
  "type": "agent_completed",
  "workflow_id": "string",
  "agent": "analyst"
}
```

## Approval Required

```json
{
  "type": "approval_required",
  "workflow_id": "string",
  "reason": "string",
  "options": [
    "continue",
    "modify"
  ]
}
```

## Workflow Completed

```json
{
  "type": "workflow_completed",
  "workflow_id": "string",
  "output_path": "outputs/proposal_reviewed.md"
}
```

- [ ] event schema を実装
- [ ] event type を centralize
- [ ] unknown event を reject / log

---

# 9. Directory Structure TODO

```text
ai_agent/
|
+-- CLAUDE.md
+-- README.md
+-- docs/
|   +-- design_doc_bot.md
|   +-- spec_todo.md
|   +-- agent_contracts.md
|   +-- development_workflow.md
|   +-- multi_bot_architecture.md
+-- .env.example
|
+-- app/
+-- agents/
+-- flows/
+-- llm/
+-- agui/
+-- prompts/
+-- examples/
+-- outputs/
+-- tests/
```

- [ ] project directory 作成
- [ ] README.md 作成
- [ ] `.env.example` 作成
- [ ] Python package structure 作成

---

# 10. Implementation TODO

## Phase 0 - Project Bootstrap

- [ ] Python environment 作成
- [ ] dependency 管理方法を決める
- [ ] `pyproject.toml` を作成
- [ ] `.gitignore` 作成
- [ ] `.env.example` 作成
- [ ] sample `examples/proposal.md` 作成
- [ ] startup command を README に記載

---

## Phase 1 - vLLM

### Goal

```text
Python -> vLLM -> Qwen
```

### TODO

- [ ] vLLM 起動方法を確認
- [ ] Qwen2.5 model を load
- [ ] `/v1/models` 確認
- [ ] `/v1/chat/completions` smoke test
- [ ] 日本語 prompt smoke test
- [ ] Python client 作成
- [ ] timeout 設定
- [ ] connection error handling

### Done Condition

- [ ] Python script から Qwen の応答を取得できる

---

## Phase 2 - CrewAI Single Agent

### TODO

- [ ] CrewAI install
- [ ] local LLM adapter 設定
- [ ] Analyst Agent 作成
- [ ] analyst prompt 作成
- [ ] sample proposal を解析
- [ ] response を console に表示
- [ ] error handling

### Done Condition

- [ ] Analyst Agent が local Qwen で動く

---

## Phase 3 - Three Agents

### TODO

- [ ] Risk Reviewer Agent 作成
- [ ] Editor Agent 作成
- [ ] prompts を分離
- [ ] Agent 間 context passing
- [ ] sequential execution
- [ ] final markdown generation

### Done Condition

```text
proposal
  |
Analyst
  |
Risk Reviewer
  |
Editor
  |
proposal_reviewed.md
```

が CLI 上で動く。

---

## Phase 4 - CrewAI Flow

### TODO

- [ ] ReviewState 定義
- [ ] CrewAI Flow 作成
- [ ] sequential states 実装
- [ ] error state 実装
- [ ] workflow_id 導入
- [ ] basic logging

### Done Condition

- [ ] state を持った workflow として完走する

---

## Phase 5 - AG-UI

### TODO

- [ ] AG-UI dependency 導入
- [ ] AG-UI server / endpoint 作成
- [ ] workflow_started event
- [ ] agent_started event
- [ ] agent_message event
- [ ] agent_completed event
- [ ] workflow_completed event
- [ ] workflow_failed event
- [ ] streaming 動作確認

### Done Condition

- [ ] 外部 UI から Agent の進捗を観測できる

---

## Phase 6 - OpenBot

### TODO

- [ ] OpenBot 起動
- [ ] AG-UI endpoint 接続
- [ ] prompt input
- [ ] proposal input
- [ ] workflow start
- [ ] Agent progress UI
- [ ] final result UI

### Done Condition

- [ ] OpenBot から workflow を実行できる

---

## Phase 7 - Human-in-the-loop

### TODO

- [ ] risk severity rule を定義
- [ ] `approval_required` 判定
- [ ] approval_required event
- [ ] workflow pause
- [ ] Continue action
- [ ] Modify action
- [ ] approval_received event
- [ ] workflow resume
- [ ] human decision を Editor に渡す

### Done Condition

```text
Risk detected
      |
workflow paused
      |
OpenBot approval
      |
workflow resumed
```

が動く。

---

## Phase 8 - Demo Polish

### TODO

- [ ] sample proposal 改善
- [ ] 重大リスクが確実に発生する demo input 作成
- [ ] workflow status wording 改善
- [ ] Markdown output layout 改善
- [ ] README demo steps
- [ ] screenshot 用シナリオ
- [ ] 5 分デモ手順作成

---

# 11. Tests

## Unit

- [ ] config test
- [ ] proposal loader test
- [ ] state validation test
- [ ] event schema test

## LLM

- [ ] vLLM connection smoke test
- [ ] invalid endpoint test
- [ ] timeout test

## Agent

- [ ] Analyst output test
- [ ] Risk Reviewer output test
- [ ] Editor output test

## Flow

- [ ] normal flow test
- [ ] approval flow test
- [ ] agent error flow test

## Integration

- [ ] OpenBot -> AG-UI
- [ ] AG-UI -> CrewAI
- [ ] CrewAI -> vLLM
- [ ] full E2E

---

# 12. MVP Acceptance Criteria

MVP は以下をすべて満たしたら完成。

- [ ] local vLLM が動作
- [ ] Qwen2 / Qwen2.5 が応答
- [ ] CrewAI から local LLM に接続
- [ ] Analyst Agent が動作
- [ ] Risk Reviewer Agent が動作
- [ ] Editor Agent が動作
- [ ] 3 Agent の workflow が完走
- [ ] AG-UI で状態イベントを送信
- [ ] OpenBot 上で進捗を確認
- [ ] approval_required を表示
- [ ] 人間が Continue / Modify を選択
- [ ] workflow が再開
- [ ] final Markdown が生成
- [ ] 外部 LLM API なしで demo 可能

---

# 13. Demo Scenario

入力:

```text
生成 AI を利用して顧客問い合わせを自動化する。
顧客の問い合わせ履歴を LLM に送り、
自動で回答を生成する。
```

期待する Risk Reviewer の指摘:

```text
顧客情報や個人情報が LLM に送信される可能性があります。
データ送信先、マスキング、保持ポリシーの確認が必要です。
```

UI:

```text
Risk Reviewer
Critical Risk Detected

[Continue]
[Modify]
```

Modify を選択した場合:

```text
local LLM
PII masking
human approval
audit log
```

などを Editor が改善案として反映する。

---

# 14. Future Backlog

MVP 後に検討する。

## Agentic Mesh

- [ ] Agent Registry
- [ ] Agent Metadata
- [ ] Agent discovery
- [ ] Dynamic Agent selection
- [ ] Agent Marketplace
- [ ] Rating / feedback

## Messaging

- [ ] Kafka
- [ ] RabbitMQ
- [ ] async execution

## Governance

- [ ] Guardrail
- [ ] Policy Engine
- [ ] RBAC
- [ ] ABAC
- [ ] audit trail

## Knowledge

- [ ] RAG
- [ ] Vector DB
- [ ] GraphRAG
- [ ] Knowledge Graph
- [ ] Ontology

## Observability

- [ ] OpenTelemetry
- [ ] Langfuse
- [ ] distributed tracing
- [ ] cost monitoring

## Platform

- [ ] Docker Compose
- [ ] Kubernetes
- [ ] GPU scheduling
- [ ] Model routing
- [ ] AI Gateway

---

# 15. Next Action

最初に行うこと:

```text
Phase 1: vLLM connection
```

具体的には:

- [ ] `vllm serve` で Qwen2.5 を起動
- [ ] curl で `/v1/chat/completions` を確認
- [ ] Python から同 API を呼ぶ
- [ ] CrewAI Single Agent を local vLLM に接続

この段階では OpenBot / AG-UI はまだ触らない。
