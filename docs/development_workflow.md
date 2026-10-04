# Development Workflow

実装順序と各 phase の Test-First Checkpoint。

> 各 phase の TODO と Done Condition は [`spec_todo.md`](./spec_todo.md) を正とする。

---

## 1. 実装順序

必ず小さく動作確認しながら進める。**複数コンポーネントを一度に実装しない。**

Phase 番号は [`spec_todo.md`](./spec_todo.md) の「10. Implementation TODO」と一致させる。

| # | Phase | 主な対象 |
|---|---|---|
| 0 | Project Bootstrap | `pyproject.toml`, `.env.example`, `examples/proposal.md` |
| 1 | vLLM | `llm/client.py` |
| 2 | CrewAI Single Agent | `agents/analyst.py`, `prompts/analyst.md` |
| 3 | Three Agents | `agents/*.py`, `prompts/*.md` |
| 4 | CrewAI Flow | `flows/proposal_review.py` |
| 5 | AG-UI | `agui/server.py`, `agui/events.py` |
| 6 | OpenBot | AG-UI endpoint 接続 |
| 7 | Human-in-the-loop | `flows/`, `agui/` |
| 8 | Demo Polish | `examples/`, `outputs/`, `README.md` |

---

## 2. Test-First Checkpoints

各 phase で最低限次を確認してから次へ進む。

| Phase | 確認内容 |
|---|---|
| 1 vLLM | `curl` / Python → `/v1/chat/completions` → response |
| 2 CrewAI Single Agent | one agent → Qwen response |
| 3 Three Agents | proposal → analyst → reviewer → editor |
| 4 CrewAI Flow | state を持った workflow として完走する |
| 5 AG-UI | Phase 5 担当の [AG-UI イベント](./agent_contracts.md#3-ag-ui-イベント) が観測できる |
| 6 OpenBot | OpenBot から workflow を実行できる |
| 7 HITL | risk detected → paused → human input → resumed（`approval_required` / `approval_received` を含む） |

---

## 3. vLLM 疎通確認例

```bash
curl -s "$VLLM_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $VLLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "'"$VLLM_MODEL"'", "messages": [{"role": "user", "content": "hello"}]}'
```
