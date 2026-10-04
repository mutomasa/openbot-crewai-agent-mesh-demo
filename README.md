# OpenBot + AG-UI + CrewAI + Local vLLM Demo

**OpenBot**、**AG-UI**、**CrewAI**、ローカル **vLLM** 上の **Qwen** を組み合わせ、
「複数の AI Agent が協調し、人間が途中で確認・承認できる」最小 Multi-Agent デモです。
最初のユースケースは **AI 提案書レビュー・チーム** です。ユーザーが Markdown の提案書を渡すと、
複数の Agent が順番に分析・レビュー・修正し、OpenBot 上で進行状況と結果を確認できます。

> **ステータス**: 設計ドキュメント段階です。アプリケーションコード（`app/`、`agents/` など）はまだ存在しません。
> 進捗は [`docs/spec_todo.md`](docs/spec_todo.md) を参照してください。

---

## デモで示すこと

- CrewAI による Multi-Agent orchestration
- AG-UI による Agent と UI 間のイベント連携
- OpenBot による Agent workspace / Human-in-the-loop
- vLLM + Qwen によるローカル LLM 実行（外部 LLM API なしでデモ可能）
- LLM Runtime と Agent Runtime の分離

---

## アーキテクチャ

```text
OpenBot ──AG-UI──▶ CrewAI Flow ──▶ OpenAI互換API ──▶ vLLM :8000 ──▶ Qwen2.5-7B-Instruct
                     │
                     ├─ Analyst
                     ├─ Risk Reviewer ──▶ HITL Event
                     └─ Editor
```

| Component | Responsibility |
|---|---|
| OpenBot | ユーザー操作、結果表示、ファイル入力、承認 UI |
| AG-UI | Agent 実行状態、streaming、tool call、HITL のイベント連携 |
| CrewAI | Multi-Agent orchestration、役割分担、実行順序 |
| vLLM | OpenAI compatible API を提供する LLM runtime |
| Qwen | 推論、文章解析、レビュー、改善案生成 |

詳細は [`docs/design_doc_bot.md`](docs/design_doc_bot.md) を参照してください。

---

## Agent 構成

| Agent | 入力 | 出力 |
|---|---|---|
| **Analyst** | proposal markdown | goal / target / problem / KPI / assumptions / missing requirements |
| **Risk Reviewer** | original proposal, analyst result | risks / severity / mitigation / approval_required |
| **Editor** | original proposal, analysis, risks, human decision | revised markdown |

初期段階では全 Agent が同じ LLM（Qwen2.5）を共有します。
入出力契約の正式な定義は [`docs/agent_contracts.md`](docs/agent_contracts.md) を参照してください。

---

## ワークフロー / Human-in-the-loop

```text
proposal ──▶ Analyst ──▶ Risk Reviewer ──▶ (HITL) ──▶ Editor ──▶ outputs/proposal_reviewed.md
```

1. Risk Reviewer は重大なリスクを検出すると `approval_required = true` を返す
2. CrewAI Flow はそこで停止し、AG-UI に `approval_required` event を送る
3. 人間が `continue`（そのまま Editor へ）または `modify`（修正指示を Editor に渡す）を選択すると再開する

AG-UI イベント（8 種）: `workflow_started` / `agent_started` / `agent_message` / `agent_completed` /
`approval_required` / `approval_received` / `workflow_completed` / `workflow_failed`
（event payload は [`docs/spec_todo.md`](docs/spec_todo.md) の「API / Event Contract」を参照）

---

## 前提条件とセットアップ

- Python 3.11+
- vLLM が動作するローカル環境
- 推奨モデル: `Qwen/Qwen2.5-7B-Instruct`（Tool Calling を利用する場合は Qwen2 より Qwen2.5 を優先）

### 1. vLLM の起動

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

Tool Calling を利用しない場合は `--enable-auto-tool-choice` と `--tool-call-parser` を省略できます。

> MVP は localhost 前提です。`0.0.0.0` で公開する場合はネットワーク制御を行ってください。

### 2. 環境変数

endpoint と model はハードコードせず、環境変数から読み込みます（`.env.example` を用意する予定・未作成）。

```bash
VLLM_BASE_URL=http://localhost:8000/v1
VLLM_API_KEY=EMPTY
VLLM_MODEL=Qwen/Qwen2.5-7B-Instruct
```

### 3. vLLM 疎通確認

```bash
curl -s "$VLLM_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $VLLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "'"$VLLM_MODEL"'", "messages": [{"role": "user", "content": "hello"}]}'
```

### 4. デモの実行（未実装）

アプリケーションの起動コマンドは未定です。Phase 0 で `pyproject.toml` と startup command を
本 README に記載する予定です（[`docs/spec_todo.md`](docs/spec_todo.md) Phase 0 / Phase 8 参照）。

---

## ディレクトリ構成（予定）

```text
.
├── CLAUDE.md
├── README.md
├── .env.example        # 予定
├── docs/               # 設計・仕様ドキュメント
├── app/                # main.py, config.py
├── agents/             # analyst.py, risk_reviewer.py, editor.py
├── flows/              # proposal_review.py
├── llm/                # client.py
├── agui/               # server.py, events.py
├── prompts/            # analyst.md, risk_reviewer.md, editor.md
├── examples/           # proposal.md
├── outputs/            # proposal_reviewed.md の出力先
└── tests/              # test_llm.py, test_agents.py, test_flow.py
```

Prompt は Python コードに埋め込まず `prompts/` に置き、Prompt 変更だけで Agent の振る舞いを調整できるようにします。

---

## 開発の進め方

小さく動作確認しながら、1 コンポーネントずつ実装します。

| # | Phase | Done Condition |
|---|---|---|
| 0 | Project Bootstrap | 環境・`pyproject.toml`・`.env.example`・sample proposal が揃う |
| 1 | vLLM | Python script から Qwen の応答を取得できる |
| 2 | CrewAI Single Agent | Analyst Agent が local Qwen で動く |
| 3 | Three Agents | proposal → Analyst → Risk Reviewer → Editor が CLI 上で動く |
| 4 | CrewAI Flow | state を持った workflow として完走する |
| 5 | AG-UI | 外部 UI から Agent の進捗を観測できる |
| 6 | OpenBot | OpenBot から workflow を実行できる |
| 7 | Human-in-the-loop | risk detected → paused → OpenBot approval → resumed |
| 8 | Demo Polish | 5 分デモ手順・demo input・README demo steps が揃う |

- 実装順序とチェックポイント: [`docs/development_workflow.md`](docs/development_workflow.md)
- TODO・Done Condition・**MVP Acceptance Criteria**（完了の基準）: [`docs/spec_todo.md`](docs/spec_todo.md)

---

## MVP スコープ外

MVP が完了するまで、以下は実装しません（正式な一覧は [`docs/spec_todo.md`](docs/spec_todo.md) NFR-003）。

- RAG / Vector DB / Knowledge Graph
- Kafka / RabbitMQ
- Kubernetes
- MCP
- OAuth / 認証 / RBAC
- Web Search / 外部 SaaS
- Agent Marketplace
- Langfuse / OpenTelemetry

---

## ドキュメント一覧

| Doc | 内容 |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Claude Code 向け作業ルール（コーディング規約、Error Policy、Logging） |
| [`docs/design_doc_bot.md`](docs/design_doc_bot.md) | 設計・アーキテクチャ・ディレクトリ構成 |
| [`docs/spec_todo.md`](docs/spec_todo.md) | 要件・TODO・MVP Acceptance Criteria（進捗の基準） |
| [`docs/agent_contracts.md`](docs/agent_contracts.md) | Agent 入出力・HITL・AG-UI イベント |
| [`docs/development_workflow.md`](docs/development_workflow.md) | 実装順序・Test-First Checkpoints |
| [`docs/multi_bot_architecture.md`](docs/multi_bot_architecture.md) | 将来の Multi-Bot 拡張（MVP 対象外） |

---

## 将来拡張

MVP 完成後、単一 Bot / 単一 Crew から、複数 Bot・複数 Crew・共有 Agent Registry・
Agent Marketplace を備えた **Agentic Mesh** の最小 PoC へ段階的に発展させます。

```text
Bot = AI社員 / Flow = 業務プロセス / Crew = AIチーム / Agent = 専門家 / LLM = 推論エンジン
```

方針は「Bot を増やす前に Agent を増やす」（Agent → Crew → Bot の順に抽象度を上げる）です。
詳細は [`docs/multi_bot_architecture.md`](docs/multi_bot_architecture.md) と
[`docs/spec_todo.md`](docs/spec_todo.md) の Future Backlog を参照してください。
