---
version: alpha
name: Agent Mesh Workspace
description: OpenBot 上で Multi-Agent の進行・Human-in-the-loop 承認・レビュー結果を扱うワークスペース UI のデザインシステム。
colors:
  primary: "#1F2937"
  on-primary: "#FFFFFF"
  secondary: "#4B5563"
  accent: "#2563EB"
  on-accent: "#FFFFFF"
  neutral: "#F8FAFC"
  surface: "#FFFFFF"
  border: "#E2E8F0"
  status-done: "#047857"
  status-done-container: "#ECFDF5"
  status-running: "#1D4ED8"
  status-running-container: "#EFF6FF"
  status-waiting: "#475569"
  status-waiting-container: "#F1F5F9"
  risk-critical: "#B91C1C"
  risk-critical-container: "#FEF2F2"
typography:
  h1:
    fontFamily: Noto Sans JP
    fontSize: 1.75rem
    fontWeight: 700
    lineHeight: 1.3
  h2:
    fontFamily: Noto Sans JP
    fontSize: 1.25rem
    fontWeight: 700
    lineHeight: 1.4
  body-md:
    fontFamily: Noto Sans JP
    fontSize: 1rem
    fontWeight: 400
    lineHeight: 1.7
  body-sm:
    fontFamily: Noto Sans JP
    fontSize: 0.875rem
    fontWeight: 400
    lineHeight: 1.6
  label-md:
    fontFamily: Noto Sans JP
    fontSize: 0.875rem
    fontWeight: 600
    lineHeight: 1.4
  mono-sm:
    fontFamily: JetBrains Mono
    fontSize: 0.8125rem
    fontWeight: 400
    lineHeight: 1.6
rounded:
  sm: 4px
  md: 8px
  lg: 12px
  full: 9999px
spacing:
  xs: 4px
  sm: 8px
  md: 16px
  lg: 24px
  xl: 40px
components:
  app-header:
    backgroundColor: "{colors.primary}"
    textColor: "{colors.on-primary}"
    typography: "{typography.h2}"
    padding: 16px
  page:
    backgroundColor: "{colors.neutral}"
    textColor: "{colors.primary}"
    typography: "{typography.body-md}"
  card:
    backgroundColor: "{colors.surface}"
    textColor: "{colors.primary}"
    rounded: "{rounded.lg}"
    padding: 24px
  card-meta:
    backgroundColor: "{colors.surface}"
    textColor: "{colors.secondary}"
    typography: "{typography.body-sm}"
  divider:
    backgroundColor: "{colors.border}"
    height: 1px
  button-primary:
    backgroundColor: "{colors.accent}"
    textColor: "{colors.on-accent}"
    typography: "{typography.label-md}"
    rounded: "{rounded.md}"
    padding: 12px
  button-secondary:
    backgroundColor: "{colors.surface}"
    textColor: "{colors.primary}"
    typography: "{typography.label-md}"
    rounded: "{rounded.md}"
    padding: 12px
  status-badge-done:
    backgroundColor: "{colors.status-done-container}"
    textColor: "{colors.status-done}"
    typography: "{typography.label-md}"
    rounded: "{rounded.full}"
  status-badge-running:
    backgroundColor: "{colors.status-running-container}"
    textColor: "{colors.status-running}"
    typography: "{typography.label-md}"
    rounded: "{rounded.full}"
  status-badge-waiting:
    backgroundColor: "{colors.status-waiting-container}"
    textColor: "{colors.status-waiting}"
    typography: "{typography.label-md}"
    rounded: "{rounded.full}"
  approval-card:
    backgroundColor: "{colors.risk-critical-container}"
    textColor: "{colors.risk-critical}"
    typography: "{typography.body-md}"
    rounded: "{rounded.lg}"
    padding: 24px
  log-stream:
    backgroundColor: "{colors.status-waiting-container}"
    textColor: "{colors.primary}"
    typography: "{typography.mono-sm}"
    rounded: "{rounded.md}"
    padding: 16px
---

## Overview

複数の AI Agent が協調して作業し、人間が要所で判断を下す「Agent ワークスペース」のための UI。
雰囲気は **落ち着いた業務ツール**。装飾よりも「いま誰が何をしていて、人間は何を判断すべきか」が
一目で分かることを最優先する。

- 主役は Agent の進行状況（Done / Running / Waiting）と、Human-in-the-loop の承認カード。
- 色は意味を持つ場面（状態・リスク・主要アクション）にだけ使い、それ以外は無彩色で構成する。
- 将来の Agentic Mesh / Agent Fleet 化で Agent や Runtime が増えても、同じ部品（カード・バッジ）の繰り返しで表現できる構造にする。

## Colors

ベースは高コントラストの無彩色。有彩色は「状態」と「行動」にのみ割り当てる。

- **Primary (#1F2937):** 本文・見出し・ヘッダー背景。インクに相当する主色。
- **Secondary (#4B5563):** メタ情報、キャプション、補足テキスト。
- **Accent (#2563EB):** 主要アクション（実行・Continue）専用。1 画面に主要ボタンは 1 つまで。
- **Neutral (#F8FAFC) / Surface (#FFFFFF) / Border (#E2E8F0):** ページ背景・カード面・区切り線。
- **Status Done (#047857):** 完了した Agent。
- **Status Running (#1D4ED8):** 実行中の Agent。Accent と同系色だが、バッジ（container 背景付き）でのみ使い、ボタンと区別する。
- **Status Waiting (#475569):** 待機中・未着手の Agent。
- **Risk Critical (#B91C1C):** Risk Reviewer が critical risk を検出し、人間の承認が必要な状態。HITL 承認カード以外では使わない。

状態は色だけで伝えない。必ずラベル文字列（`Done` / `Running` / `Waiting`）を併記する。

## Typography

日本語 UI を前提に **Noto Sans JP** を使い、ログ・ID・ファイル名には **JetBrains Mono** を使う。

- **h1:** 画面タイトル（例: `Proposal Review`）。
- **h2:** セクション見出し、ヘッダー内タイトル。
- **body-md:** 本文。提案書のレビュー結果など長文を読むため行間は広め（1.7）。
- **body-sm:** メタ情報（latency、workflow_id、タイムスタンプ）。
- **label-md:** ボタン、ステータスバッジ。
- **mono-sm:** AG-UI イベントログ、ファイル名（`proposal_reviewed.md`）。

## Layout

8px グリッド。`spacing` トークン（xs 4px 〜 xl 40px）以外の余白は使わない。

- デスクトップは 2 カラム: 左に Workflow（Agent ごとのステータス一覧）、右に詳細（出力・承認・結果）。
- 幅 768px 未満では 1 カラムに縦積みし、承認カードを最上部に出す。
- 本文の最大幅は 72 文字程度（約 720px）に抑え、長文レビュー結果を読みやすくする。

## Elevation & Depth

影はほぼ使わない。階層は背景色の差（Neutral → Surface）と 1px の Border で表現する。
例外として、HITL 承認カードのみ注意を引くために弱い影（`0 1px 3px rgba(0,0,0,0.08)`）を許可する。

## Shapes

- カード: `rounded.lg`（12px）
- ボタン・ログ枠: `rounded.md`（8px）
- ステータスバッジ: `rounded.full`（pill 形状）
- 入力欄: `rounded.sm`（4px）

## Components

- **app-header:** 濃色ヘッダー。アプリ名と現在の workflow_id を表示。
- **card / card-meta:** Agent 1 体 = カード 1 枚。カード上部に Agent 名とステータスバッジ、下部に latency などのメタ情報。
- **status-badge-done / -running / -waiting:** Agent の状態表示。AG-UI の実行状態イベントと 1 対 1 に対応させる。
- **approval-card:** critical risk 検出時の HITL 承認 UI。リスク内容を本文で示し、`Continue`（button-primary）と `Modify`（button-secondary）を並べる。
- **button-primary / button-secondary:** 主要アクションと副次アクション。破壊的・不可逆な操作は primary にしない。
- **log-stream:** AG-UI イベントや Agent のストリーミング出力を等幅フォントで表示。
- **divider:** セクション区切り。

## Do's and Don'ts

- **Do:** 状態は「色 + ラベル + （必要なら）アイコン」で冗長に伝える。
- **Do:** テキストと背景の組み合わせは WCAG AA（4.5:1）以上を守る（`npx @google/design.md lint DESIGN.md` で確認）。
- **Do:** Agent が増えてもカードの繰り返しで済むよう、Agent 固有の色やレイアウトを作らない。
- **Don't:** Risk Critical の赤を装飾やブランドカラーとして使わない。
- **Don't:** LLM prompt 全文や機密情報を UI 上のログにそのまま表示しない（Logging ポリシーに準拠）。
- **Don't:** トークンにない色・余白・角丸をコードに直書きしない。追加が必要なら先に DESIGN.md を更新する。
