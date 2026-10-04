# CLAUDE.md

**OpenBot + AG-UI + CrewAI + local vLLM/Qwen** による Multi-Agent デモ。
最初のユースケースは **AI 提案書レビュー・チーム**（Analyst → Risk Reviewer → HITL → Editor）。

```text
OpenBot ──AG-UI──▶ CrewAI Flow ──▶ OpenAI互換API ──▶ vLLM :8000 ──▶ Qwen2.5-7B-Instruct
```

## 📚 Authoritative Docs

作業前に必ず確認する。設計変更が必要な場合は **docs を先に更新してから** コードを変更する。

| Doc | 内容 |
|---|---|
| [`docs/design_doc_bot.md`](docs/design_doc_bot.md) | 設計・アーキテクチャ・ディレクトリ構成 |
| [`docs/spec_todo.md`](docs/spec_todo.md) | 要件・TODO・**MVP Acceptance Criteria**（進捗の基準） |
| [`docs/agent_contracts.md`](docs/agent_contracts.md) | Agent 入出力・HITL・AG-UI イベント |
| [`docs/development_workflow.md`](docs/development_workflow.md) | 実装順序・Test-First Checkpoints |
| [`docs/multi_bot_architecture.md`](docs/multi_bot_architecture.md) | 将来の Multi-Bot 拡張（MVP 対象外） |
| [`DESIGN.md`](DESIGN.md) | UI デザインシステム（[Google DESIGN.md 形式](https://github.com/google-labs-code/design.md)）。UI 実装時は必ずこのトークンに従う |

## 🚫 Scope (MVP)

`spec_todo.md` の MVP が完了するまで、以下は **実装しない**（正: NFR-003）。

> RAG / Vector DB / Knowledge Graph / Kafka / RabbitMQ / Kubernetes / MCP /
> OAuth / RBAC / Web Search / 外部 SaaS / Agent Marketplace / Langfuse / OpenTelemetry

## ⚙️ LLM Configuration

endpoint・model は **ハードコード禁止**。環境変数から読む（`app/config.py`）。

```bash
VLLM_BASE_URL=http://localhost:8000/v1
VLLM_API_KEY=EMPTY
VLLM_MODEL=Qwen/Qwen2.5-7B-Instruct
```

## 🧑‍💻 Coding Rules

### Python

- Python **3.11+**、type hint 必須
- 状態定義は `dataclass` または Pydantic
- import side effect / global mutable state を使わない
- error handling を省略しない

### 構造

- 1 ファイル **< 300 行** を目安
- 関数は単一責任（`run_everything()` ではなく `load_proposal()` → `run_analysis()` → `run_risk_review()` → `request_approval()` → `run_editor()` → `save_result()`）
- レイヤ: `app/` · `agents/` · `flows/` · `llm/` · `agui/` · `prompts/` · `examples/` · `outputs/` · `tests/`

### Prompt

- Prompt を Python コードに長文で埋め込まない
- `prompts/{analyst,risk_reviewer,editor}.md` に置き、**Prompt 変更だけで振る舞いを調整できる**ようにする

### Error Policy

- 黙って fallback しない。明示的な例外を投げる
  - `VLLMConnectionError` / `AgentExecutionError` / `InvalidProposalError` / `ApprovalTimeoutError`
- retry は最大 **1〜2 回**。複雑な retry logic は入れない

### Logging

- 標準 `logging` を使用
- 必須項目: `timestamp` / `workflow_id` / `agent` / `event` / `latency` / `error`
- LLM prompt 全文や機密情報はデフォルトで **ログに出さない**

### Dependencies

新しい package を追加する前に確認する:

1. 標準ライブラリで代替できないか
2. CrewAI / AG-UI に既存機能がないか
3. 本当に MVP に必要か

## 🔁 Working Style

1. `design_doc_bot.md` と `spec_todo.md` の未完了項目を確認
2. 最小単位で実装（[実装順序](docs/development_workflow.md#1-実装順序) に従う）
3. test / smoke test を実行
4. 完了した TODO を `spec_todo.md` で更新
5. 次の最小タスクへ

- 大規模な rewrite を避ける
- 動いている既存コードへの破壊的変更を避ける
- **Definition of Done**: `spec_todo.md` の MVP Acceptance Criteria をすべて満たすこと
