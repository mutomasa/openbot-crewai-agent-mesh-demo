# Multi-Bot Architecture
## OpenBot + AG-UI + CrewAI + Local vLLM

## 1. 目的

本ドキュメントでは、単一の OpenBot Bot と単一 CrewAI Crew から始めて、将来的に複数 Bot、複数 Crew、共有 Agent Registry、Agent Marketplace へ拡張するためのアーキテクチャを整理する。

基本概念は以下。

- **Bot = AI社員 / サービス境界**
- **Flow = AI社員が仕事を進める業務プロセス**
- **Crew = AIチーム**
- **Agent = チーム内の専門家**
- **LLM = Agent の推論エンジン**

重要なのは、Bot と Crew を 1:1 に固定しないことである。1つの Bot が、タスクに応じて複数の Crew を使い分けることができる。

---

## 2. Conceptual Model

```text
User
  |
  v
Bot
  |
  v
Flow
  |
  v
Crew
  |
  v
Agent
  |
  v
LLM
```

| Layer | Meaning | Example |
|---|---|---|
| Bot | ユーザーから見えるAI社員 / サービス | Proposal Review Bot |
| Flow | 業務プロセス / タスク制御 | Proposal Review Flow |
| Crew | 仕事を担当するAIチーム | Technology Review Crew |
| Agent | チーム内の専門家 | Security Agent |
| LLM | 推論エンジン | Qwen2.5 via vLLM |

---

## 3. Bot と Crew の関係

Bot は「AI社員」としてユーザーに見える。Crew は Bot の内部で特定の仕事を担当する「AIチーム」である。

```text
Bot = AI社員
Crew = AI社員が利用する専門チーム
```

Bot が Crew の上司という意味ではない。Bot はユーザーとの境界となる Persona / Service Boundary であり、Flow がタスクを解釈し、必要な Crew を選択する。

---

## 4. One Bot / Multiple Crews

```text
Proposal Review Bot
        |
        v
Proposal Review Flow
        |
        +-------------------+
        |                   |
        v                   v
 Business Crew       Technology Crew
        |                   |
        |                   +-- Architect Agent
        |                   +-- Security Agent
        |
        +-- Business Analyst
        +-- Cost Analyst
        |
        +-------------------+
                |
                v
          Editorial Crew
                |
                +-- Reviewer
                +-- Editor
```

ユーザーからは 1 人の AI 社員に見えるが、内部では複数の Crew が連携する。

---

## 5. Bot = Persona / Crew = Capability

Bot は Persona / Service Boundary を表し、Crew は Capability を表す。

```text
Development Bot
      |
      v
Development Flow
      |
      +-- Architecture Crew
      +-- Coding Crew
      +-- QA Crew
      +-- Security Crew
```

ユーザー入力に応じて Flow が Crew を切り替える。

```text
「設計レビューして」 -> Architecture Crew
「バグを直して」     -> Coding Crew
「脆弱性を確認して」 -> Security Crew
「リリース判定して」 -> QA Crew + Security Crew
```

---

## 6. Multiple Bots

Bot を増やすと、ユーザーから見える AI 社員を分けられる。

```text
OpenBot
  |
  +-- Proposal Review Bot
  +-- Development Bot
  +-- SRE Bot
  +-- Research Bot
```

---

## 7. Multiple Bots + Multiple Crews

```text
OpenBot
|
+-- Proposal Review Bot
|     |
|     v
|   Proposal Review Flow
|     |
|     +-- Business Crew
|     +-- Technology Crew
|     +-- Editorial Crew
|
+-- Development Bot
|     |
|     v
|   Development Flow
|     |
|     +-- Architecture Crew
|     +-- Coding Crew
|     +-- QA Crew
|
+-- SRE Bot
      |
      v
    Incident Flow
      |
      +-- Monitoring Crew
      +-- Diagnosis Crew
      +-- Recovery Crew
```

---

## 8. Role of AG-UI

AG-UI は Bot と Agent Runtime の境界に置く。

```text
OpenBot
   |
   | AG-UI
   v
Agent Runtime
   |
   +-- CrewAI Flow
   +-- Crew
   +-- Agents
```

AG-UI の責務:

- prompt / message
- workflow state
- agent state
- streaming
- tool call
- human approval
- interruption / resume
- final result

責務分離は以下。

```text
OpenBot = UX / Workspace
AG-UI   = Agent UI Protocol
CrewAI  = Agent Orchestration
vLLM    = Model Runtime
```

---

## 9. Bot Routing

### 9.1 Explicit Selection

```text
User
 |
 v
OpenBot
 |
 +-- Development Bot
 +-- SRE Bot
 +-- Research Bot
```

MVP ではユーザーが Bot を直接選ぶ方式が簡単。

### 9.2 Automatic Routing

```text
User
 |
 v
OpenBot
 |
 v
Bot Router
 |
 +----------+----------+
 |          |          |
 v          v          v
Dev Bot    SRE Bot   Research Bot
```

例:

```text
「昨日からAPIが遅い」
        |
        v
Bot Router
        |
        v
SRE Bot
```

---

## 10. Crew Routing

Bot 内でも Crew Router を持てる。

```text
Development Bot
      |
      v
Development Flow
      |
      v
Crew Router
      |
      +--------------+--------------+
      |              |              |
      v              v              v
Architecture      Coding         Security
Crew              Crew           Crew
```

```text
User intent
   |
Bot selection
   |
Task decomposition
   |
Crew selection
```

---

## 11. Agent Registry

Agent 数が増えると、Crew のコードに Agent を固定する方式では管理しづらくなる。そこで Agent Registry を導入する。

```text
Agent Registry
+--------------------------------------------+
| Agent ID                                   |
| Name                                       |
| Capability                                 |
| Input Schema                               |
| Output Schema                              |
| Model                                      |
| Tools                                      |
| Cost                                       |
| Latency                                    |
| Trust Level                                |
| Version                                    |
| Owner                                      |
+--------------------------------------------+
```

例:

```yaml
id: security-reviewer-v1
name: Security Reviewer
capability:
  - security_review
  - privacy_review
model: qwen2.5-7b
tools:
  - file_reader
trust_level: high
```

---

## 12. Shared Agent Pool

```text
             Agent Registry
                  |
       +----------+----------+
       |          |          |
       v          v          v
 Security     Cost       Research
  Agent       Agent        Agent
       ^          ^          ^
       |          |          |
+------+---+ +----+----+ +---+------+
|          | |         | |          |
Proposal   Dev        SRE        Research
Crew       Crew       Crew       Crew
```

複数 Crew から共通 Agent を利用できるため、重複実装を減らせる。

---

## 13. Agent Marketplace

Agent Registry に発見・評価・共有機能を追加すると Agent Marketplace に発展する。

```text
Agent Marketplace
      |
      +-- Search
      +-- Discover
      +-- Install
      +-- Rate
      +-- Feedback
      +-- Version
      +-- Trust
```

---

## 14. Dynamic Crew Formation

```text
User Request
    |
    v
Planner
    |
    v
Capability Requirements
    |
    +-- business_analysis
    +-- security_review
    +-- cost_estimation
    |
    v
Agent Registry
    |
    +-- Business Agent
    +-- Security Agent
    +-- Cost Agent
    |
    v
Dynamic Crew
```

Crew を固定チームではなく Task-specific Team として生成できる。

---

## 15. Multi-Bot + Agentic Mesh

```text
                         OpenBot
                            |
                 +----------+----------+
                 |                     |
                 v                     v
           Development Bot        SRE Bot
                 |                     |
                 +----------+----------+
                            |
                         AG-UI
                            |
                            v
                       Bot Router
                            |
                            v
                    Agentic Mesh Layer
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
      Crew Router      Agent Registry     Policy Engine
          |                 |                 |
          +-----------------+-----------------+
                            |
                            v
                      Agent Runtime
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          CrewAI           CrewAI         CrewAI
           Crew             Crew           Crew
             |              |              |
             +--------------+--------------+
                            |
                            v
                        AI Gateway
                            |
                    +-------+-------+
                    |               |
                    v               v
                 local vLLM       Cloud LLM
                    |
                    v
                  Qwen
```

---

## 16. Recommended Evolution

### Phase 1 (MVP)

```text
1 Bot
  |
1 Flow
  |
1 Crew
  |
3 Agents
```

例:

```text
Proposal Review Bot
  |
Proposal Review Flow
  |
Proposal Review Crew
  |
+-- Analyst
+-- Risk Reviewer
+-- Editor
```

MVP 完了後、Agent を増やす場合は以下のような構成を検討する（MVP 対象外）。

```text
+-- Planner
+-- Analyst
+-- Security
+-- Cost
+-- Editor
```

### Phase 2

```text
1 Bot
  |
1 Flow
  |
Multiple Crews
```

```text
Proposal Review Bot
  |
Proposal Review Flow
  |
  +-- Business Crew
  +-- Technology Crew
  +-- Editorial Crew
```

### Phase 3

```text
Multiple Bots
  |
Multiple Flows
  |
Multiple Crews
```

```text
OpenBot
 |
 +-- Proposal Bot
 +-- Development Bot
 +-- SRE Bot
```

### Phase 4

```text
Multiple Bots
   |
Bot Router
   |
Multiple Crews
   |
Agent Registry
```

### Phase 5

```text
Multiple Bots
   |
Agentic Mesh
   |
Agent Registry
   |
Agent Marketplace
   |
Dynamic Crew Formation
```

---

## 17. MVP Recommendation

今回の PoC では以下を推奨する。

```text
OpenBot

Proposal Review Bot
       |
       v
Proposal Review Flow
       |
       v
Proposal Review Crew
       |
       +-- Analyst Agent
       +-- Risk Reviewer Agent
       +-- Editor Agent
       |
       v
     vLLM
       |
       v
    Qwen2.5
```

最初は Bot を 1 個、Crew も 1 個、Agent は 3 個（[`spec_todo.md`](./spec_todo.md) の MVP 定義）にする。
Planner / Security / Cost などへの Agent 分割は MVP 完了後の拡張とする。

---

## 18. Second Demo

```text
1 Bot
  |
Multiple Crews
```

```text
Proposal Review Bot
       |
Proposal Review Flow
       |
       +-- Business Crew
       +-- Technology Crew
       +-- Editorial Crew
```

これにより「AI社員が、案件に応じて複数の専門チームを使い分ける」デモができる。

---

## 19. Third Demo

次に Bot を増やす。

```text
OpenBot
 |
 +-- Proposal Review Bot
 +-- Development Bot
 +-- SRE Bot
```

ここで、

```text
Bot = AI Employee
Crew = AI Team
Agent = AI Specialist
```

という組織モデルを視覚的に示せる。

---

## 20. Future Enterprise Model

```text
Human Organization
        |
        v
AI Employees
        |
        v
AI Teams
        |
        v
AI Specialists
```

技術的には以下に対応する。

```text
Bot
 |
Flow
 |
Crew
 |
Agent
 |
LLM / Tool
```

Agentic Mesh を導入すると、

```text
Bot
 |
Flow
 |
Agentic Mesh
 |
+-- Registry
+-- Routing
+-- Marketplace
+-- Policy
+-- Guardrail
+-- Observability
 |
Crew / Agent
 |
LLM
```

という企業向け AI Agent Platform に発展させられる。

---

## 21. Key Design Principle

```text
Bot を増やす
= ユーザーから見える AI 社員 / サービスを増やす

Crew を増やす
= AI 社員が使える AI チーム / Capability を増やす

Agent を増やす
= チーム内の専門能力を増やす
```

したがって、今回の PoC ではまず

```text
Bot を増やす前に Agent を増やす
```

のが適切。

その後、

```text
Agent
  ↓
Crew
  ↓
Bot
```

の順に抽象度を上げながら拡張する。

最終的には、

```text
Multiple Bots
   +
Multiple Crews
   +
Shared Agent Registry
   +
Agent Marketplace
```

を Agentic Mesh の基盤とする。
