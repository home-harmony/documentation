# 0018. AI-Native Modular Platform Evolution, Tiered Cost Funnel, and Declarative SDUI

* **Status**: Accepted
* **Date**: 2026-11-14
* **Deciders**: Project Owner / AI Architecture Advisor

---

## Context and Problem Statement

While **FamilyLedger** began as a standalone serverless family financial hub on Rust and Amazon Aurora DSQL, the long-term vision for **Home Harmony** is a modular, AI-native personal and family life platform (**Umbrella Life OS**) encompassing finances (`FamilyLedger` as the first plugin), language learning, and lifestyle/adaptive planning.

However, building an AI-first system presents four major engineering and product risks:
1. **Token Cost & Idle Spend**: Invoking LLMs on every database event, screen load, or button click creates unpredictable token bills and high latency (2–5 seconds per interaction).
2. **Financial Determinism & Safety**: LLMs cannot be trusted to perform monetary arithmetic (`rust_decimal::Decimal`), enforce RBAC (`Role::Child` restrictions), or generate arbitrary unvalidated HTML/JS interfaces.
3. **UX Friction of Chat-Only Interfaces**: Users do not want to type long text prompts in a chat box for routine daily actions; they need tactile, point-and-click / tap-friendly controls (buttons, sliders, confirmation cards, visual charts).
4. **Time-to-Market (Scope Creep)**: Attempting to ship all domains and full generative AI capabilities simultaneously delays the initial MVP launch of `FamilyLedger`.

We need an architecture that enables a fast deterministic MVP launch (`$0.00` token cost), seamlessly layers **AWS Bedrock** and **MCP (Model Context Protocol)** on top without UI rewrites, and scales across **Mobile (Flutter)** and **Web + Admin (Angular)**.

---

## Decision Drivers

* **Zero-Token Operational Mode (`AI = OFF`)**: The entire application must function 100% deterministically with `$0.00` LLM cost when AI is disabled or when monthly token budgets are reached.
* **Tactile Declarative UX (Not Simple Chat)**: AI interactions must render structured, interactive UI cards (buttons, sliders, category pickers) rather than plain text chat replies.
* **Strict Financial Integrity**: All monetary calculations, ledger mutations, and RBAC checks must execute in deterministic Rust code (`domain/`), never inside an LLM.
* **Fast Phased Delivery**: Launch the deterministic `FamilyLedger` core first (Phase 1.0), activate AWS Bedrock SDUI cards second (Phase 1.1), add AI-generated chart specs third (Phase 1.2), and expand to new domain plugins later (Phase 2.0+).
* **Unified Multi-Platform Primitives**: Share the same declarative UI component contracts across **Mobile (Flutter)** and **Web + Admin Control Plane (Angular)**.

---

## Considered Options

1. **Option 1: Chat-First Always-On Agent**
   * Every user action and background DB event passes through an LLM orchestrator that replies in a conversational chat thread and calls backend tools.
2. **Option 2: Full Server-Driven UI (100% Backend-Rendered JSON Screens)**
   * Both standard screens and AI responses are fully dynamically generated as JSON trees from the backend on every navigation.
3. **Option 3: Phased Dual-Surface Architecture (REST + MCP) with Tiered Cost Funnel and Typed SDUI Action Cards**
   * Native/standard views for instant navigation (`$0`), shared typed SDUI cards (`TransactionDraftCard`, `BudgetRebalanceCard`, `DebtScenarioCard`, `ApprovalRequestCard`) populated either deterministically (Phase 1.0) or by **AWS Bedrock** on explicit user request (Phase 1.1), backed by self-learning categorization rules.

---

## Decision Outcome

Chosen option: **Option 3 (Phased Dual-Surface Architecture with Tiered Cost Funnel and Typed SDUI Action Cards)**.

### Key Architectural Pillars:

1. **Phased Rollout Strategy (Deterministic Core First)**:
   * **Phase 1.0 (MVP Launch — `AI = OFF`, `$0` tokens)**: Launch `FamilyLedger` with deterministic REST endpoints, Level 1 categorization rules (`ledger_categorization_rules`), pre-configured Web analytics charts, and the Web Admin Control Plane.
   * **Phase 1.1 (AI Layer Activation)**: Enable **AWS Bedrock** (Converse API with strict JSON Schema / Tool Use) and MCP tools to populate the exact same SDUI cards upon explicit user trigger.
   * **Phase 1.2 (Post-MVP AI Charts)**: Enable AI-generated `ChartCardSpec` declarations in Web UI with a **`Pin to Dashboard`** action that converts AI chart specs into permanent `$0` deterministic widgets.
   * **Phase 2.0+ (Umbrella Plugins)**: Add `Language Learning` and `Lifestyle & Adaptive Planning` modules using the same Dual-Surface (`REST + MCP`) and SDUI catalog patterns.

2. **Three-Tiered Cost Funnel**:
   * **Tier 0 (`$0` — Direct REST Execution)**: Every button click, slider adjustment, form submission, and card confirmation calls deterministic Rust REST endpoints directly without invoking an LLM.
   * **Tier 1 (`$0` — Deterministic Rules & Rust Math)**: Merchant/MCC categorization rules (`ledger_categorization_rules`), budget envelope aggregations, and *Avalanche/Snowball* debt payoff calculations run in pure Rust and SQL.
   * **Tier 2 (AWS Bedrock Tokens — Explicit Trigger Only)**: Bedrock is invoked **only** when the user explicitly triggers "Smart Input / Ask AI" (or a scheduled weekly digest)—never automatically on database CDC events.

3. **Self-Learning Rule Graduation (`AI -> Rule`)**:
   * When a user confirms a `TransactionDraftCard` (whether entered manually or parsed by AI from voice/text/receipt), the merchant-to-category mapping is persisted to `ledger_categorization_rules` so all future occurrences resolve at **Tier 1 for `$0`**.

4. **Platform Split (Flutter Mobile vs. Angular Web + Admin)**:
   * **Mobile UI (Flutter)**: Streamlined daily family companion for 3-second transaction entry, envelope monitoring, and tapping SDUI action cards.
   * **Web UI (Angular)**: Full functional parity with Mobile **plus** large-screen analytical visualizations (`Sankey/Waterfall Cashflow`, `Income vs Expense`, `Category Treemap/Donut`, `Envelope Bullet Charts`, `Debt Payoff Timeline`) **and** the **Admin Control Plane** (RBAC & `Role::Child` governance, categorization rules editor, ledger amendment inspector, module registry, and AI token budget controls in `family_system_settings`).

---

## Pros and Cons of the Options

### Option 1: Chat-First Always-On Agent
* **Positive**: Minimal structured UI design upfront.
* **Negative**: Slow (2–5s per interaction), high token burn, poor ergonomics for daily expense tracking, high risk of hallucinated numbers.

### Option 2: Full Server-Driven UI Everywhere
* **Positive**: Backend can alter any screen layout without app store releases.
* **Negative**: Excessive complexity for a first AI project; degrades offline support, animations, and numeric keypad responsiveness on mobile.

### Option 3: Phased Dual-Surface + Tiered Cost Funnel + Typed SDUI Cards
* **Positive**: Fast MVP launch with `$0` token cost; zero LLM math errors; instant REST button clicks; automatic graduation of AI insights into free deterministic rules; unified AWS IAM security via AWS Bedrock.
* **Negative**: Requires maintaining versioned JSON contracts for the SDUI card catalog across Rust, Angular (TypeScript), and Flutter (Dart).

---

## Compliance & Invariants

1. **Zero-Trust Context Propagation**: `family_id`, `user_id`, and `role` must **never** be accepted from LLM tool arguments; they are always injected from verified Cognito JWT + `X-Family-Id` membership (`ADR-0011`).
2. **No LLM Monetary Math**: Prompts and MCP tools must never ask AWS Bedrock to sum transactions or compute interest schedules; Bedrock only extracts intent parameters or selects a `ChartCardSpec` query, while Rust (`rust_decimal::Decimal`) and Aurora DSQL compute all numbers.
3. **No Implicit Background AI Calls**: Database mutations and CDC streams must never trigger AWS Bedrock calls unless explicitly configured as a scheduled summary job within the family's active `ai_monthly_limit_usd`.
4. **Direct REST Actions on SDUI Cards**: The `primary_action` of every mutating SDUI card (`TransactionDraftCard`, `BudgetRebalanceCard`, `DebtScenarioCard`, `ApprovalRequestCard`) must target a deterministic REST endpoint.
5. **Reference Specification**: Detailed card schemas, chart specifications, and admin views are maintained in [ai-native-platform-rethinking.md](../architecture/ai-native-platform-rethinking.md).
