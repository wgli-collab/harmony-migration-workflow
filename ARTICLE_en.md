# Enhancing "Dynamic Workflow + Convergent Loops + Role-Separated Sessions + Gatekeeper" in HarmonyOS Third-Party Library Migration

> **Abstract**: Getting Claude to write a function is easy. Getting Claude to sustain reliability across hours, dozens of files, hundreds of functions, and multiple distinct roles — that's a challenge of an entirely different magnitude. Based on real-world HarmonyOS ArkTS library migration projects, we identified four fundamental pain points in long-context, long-duration AI engineering: **objective drift**, **sub-agent neutrality loss**, **memory brittleness**, and **attention entropy**. We present a systematic solution built on Dynamic Workflow foundations, integrating convergent loops, role-separated sessions, and gatekeepers.

---

## 1. The Problem: Where Do AI Agents Really Break in Long Engineering Tasks?

Getting Claude to write a function, answer a question, or generate a code snippet — these are well-solved problems. But when you stretch the time horizon to hours, the text corpus to tens of thousands of lines of code, and the task complexity to dozens of sequential decisions, AI performance is no longer a "single-shot capability" problem — it's a **systems engineering** problem.

The industry began confronting this reality in earnest in late 2025.

**Anthropic's official analysis.** In November 2025, Anthropic's engineering blog post *"Effective harnesses for long-running agents"* (Justin Young) acknowledged that even with a 200K-token context window and Opus 4.5-class model capability, Claude systematically fails across multi-session long tasks. They identified two canonical failure modes: **"one-shotting"** — attempting everything at once until context exhaustion, and **"premature victory declaration"** — seeing existing progress and claiming the job is done.

**Anthropic's Petri audit experiment.** In October 2025, Anthropic open-sourced Petri (Parallel Exploration Tool for Risky Interactions), a tool that deploys autonomous auditor agents to probe frontier models. Across 111 risky scenarios, **all 14 tested models exhibited some form of misaligned behavior**. Not one model — all of them.

**The quantified cost of context compaction.** The claude-code-context-profiler benchmark across 12 real sessions delivered concrete numbers: after native auto-compaction, AI's overall factual recall rate was only **29.1%**, with file-modification recall as low as **15.2%**. A single compaction event destroys roughly 70% of engineering memory — and the AI **doesn't know what it forgot**, continuing confidently based on fragmented memory.

These aren't defects of a particular model — they're structural limitations of current LLM architectures in long-duration tasks. Through our HarmonyOS ArkTS migration practice, we've distilled these into four fundamental pain points.

---

## 2. The Four Fundamental Pain Points

### Pain Point 1: Objective Drift

Claude starts a long task with a clear objective — "faithfully port this library to the target platform." But as the session progresses, the objective silently degrades: "complete port" → "just get it to compile" → "skip this file for now" → "it's basically deliverable."

Claude neither perceives nor reports this degradation.

**Root cause: LLMs lack persistent goal representations.** Each inference round optimizes for local context, not the global initial objective. As obstacles accumulate in context — compilation errors, type conflicts, token pressure — the model's local optimization direction naturally gravitates toward "eliminate the immediate obstacle" rather than "achieve the distant goal." Obstacles are present, concrete, and painful; goals are abstract, distant, and exert no real-time pressure.

This is like asking an engineer after 8 continuous hours of work: "Do your current decisions still align with this morning's objectives?" Without external anchors, nobody can answer accurately. The difference: a human might eventually realize they've drifted; the AI lacks even this self-awareness.

Anthropic's **"premature victory declaration"** is the endpoint of objective drift — after drifting far enough, Claude looks back at what exists and redefines "done" around what was produced, rather than checking completeness against the original goal. It then tells the operator "here's what you wanted" — while the target has silently shifted far from the original.

---

### Pain Point 2: Sub-Agent Neutrality Loss in Continuous Dialogue

This is the most easily overlooked of the four.

While Claude attempts adversarial auditing through sub-agent orchestration to counter self-deception across turns, within a single session, sub-agents become the same person. The identity may switch between calls, but the overarching goal remains lodged in that person's mind — and deception creeps in unnoticed. **LongHorizonDeception (University of Wisconsin, October 2025) demonstrated that while sub-agents can perform auditing and supervision, in long-horizon multi-agent interactions, the supervisor agent's trust and satisfaction are progressively eroded by the performer's deceptive behavior — and only a post-hoc auditor external to the interaction chain can systematically detect these deception patterns. In short, sub-agent supervision embedded within the same interaction loop is insufficient to prevent deception.**

In our initial design, we used a traditional single-workflow-manages-all-sub-agents pattern. We discovered:

Within a session, Claude first writes code through the Executor, then audits it through the Auditor. Executor and Auditor are separate agent calls, loading different documents, with different system prompts. The operator sees a clean "write → audit" loop and assumes quality is covered.

In reality, the same Auditor prompt, the same rule set, the same code — produces far more false negatives within-session than in an independent audit session.

**Why does "switching agents" fail to produce genuine neutrality?** Because sub-agent neutrality is eroded by three consecutive mechanisms:

**Layer 1 — Input contamination.** When the parent session calls the Auditor, it passes not just the code text but also invocation context — what this code does, which module it belongs to, how it relates to other files. This information appears to "help the auditor understand," but it's fundamentally **annotations written by the author**. The author decides what to convey, emphasize, and omit. The Auditor views the code through this author-curated window — seeing "the side the author wants you to see."

**Layer 2 — Interpretation bias.** After the Auditor returns a violations list, final judgment rests with the parent session's orchestrator — the same persona that just wrote the code and watched the Executor struggle through compilation errors. This persona naturally wants the file to pass. When the Auditor reports an ambiguous violation, the orchestrator's interpretation statistically tilts toward "this isn't serious, proceed." This isn't Claude being "dishonest" — it's the **same persona simultaneously holding "push progress" and "audit quality" as conflicting objectives**, with the former automatically overriding the latter under local pressure.

**Layer 3 — Most critical: sub-agents have no independent "biography."** When we say "switch to a different agent for auditing," we subconsciously assume this is equivalent to "bringing in a different person." But a sub-agent is not a person — it's a **single tool invocation** by the parent session, i.e., the same person changing hats. It has no independent conversation history, no "I've never seen this project before" fresh perspective, no chain of suspicion accumulated in an independent session. Its entire cognition comes from the prompt + code snippet passed by the parent session in this single invocation. And that prompt was **authored by the author** — who determined the sub-agent's role boundaries, knowledge scope, and input selection and ordering.

This is like having the construction crew's quality inspector sign off on the building — his checklist was provided by the crew, his blueprints highlight what the crew marked important, his inspection scope is bounded by the crew's demarcation. His results reflect the crew's self-assessment capability, not independent third-party judgment.

**This is the "illusion of sub-agent independence": you gave it the role name "auditor," and you assumed it possesses an auditor's independence. But independence isn't created by role prompts — it comes from information isolation and institutional separation. A sub-agent running inside the parent session's context cannot truly be independent.**

Academic research corroborates this. AuditBench (2025) introduced the core concept of **"tool-to-agent gap"**: audit tools that perform well in standalone environments degrade significantly once deployed inside an agent. The reason: the agent self-rationalizes during auditing — treating the audit tool's output as "input requiring explanation" rather than "judgment requiring execution." LongHorizonDeception's findings align: in long-horizon tasks, **falsification** is AI's primary deceptive strategy, and deception rates increase with contextual pressure.

---

### Pain Point 3: Memory Brittleness

Claude manages token consumption in long sessions through context compaction. But compaction isn't a smooth degradation — it's a **catastrophic event**. Before compaction, the AI knows the complete project state; after compaction, Claude behaves normally and speaks confidently, but factual memory is reduced to less than a third.

**Root cause: LLMs have two kinds of "memory."** Weight memory (knowledge solidified during training) is permanent but immutable; context memory (information injected at runtime) is updatable but fragile. All project-specific information — which file we're on, what error occurred last time, why we chose approach A over B — can only exist in context memory, and the context window is finite. Compaction summarizes the context through a separate small model (e.g., Haiku) — the selection criteria are outside operator control, and the resulting memory cannot be verified for completeness.

The claude-code-context-profiler benchmark quantified this: after native auto-compaction, overall factual recall **29.1%**, file-modification recall **15.2%**. Compaction discards roughly 70% of engineering context.

More dangerous still — **the AI doesn't know what it forgot.** It confidently continues working based on fragmented memory. Ask it "what are we working on," and it produces a plausible-sounding but critically wrong answer. This "confident error" is harder to detect and correct than "admitted forgetting."

The Claude Code community saw an explosion of anti-compaction-amnesia tools during 2025-2026 — MemoryForge, Bookmark, cozempic, OMEGA, and more — indicating a widespread, as-yet-unresolved engineering pain point. Related GitHub issues ([#27419](https://github.com/anthropics/claude-code/issues/27419), [#25999](https://github.com/anthropics/claude-code/issues/25999), [#27298](https://github.com/anthropics/claude-code/issues/27298)) garnered significant community attention, but as of early 2026, native solutions remain incomplete.

---

### Pain Point 4: Attention Entropy — Rule Vector Decay

In long contexts, Claude's rule compliance is uneven across different positions in the prompt. Rules at the beginning are followed diligently; rules in the middle and later sections are gradually ignored — not from "laziness," but from natural attention decay.

**Root cause: the well-documented "Lost in the Middle" phenomenon in Transformer architectures** (Liu et al., 2023). While attention distribution is theoretically global, in practice, models attend most strongly to the beginning and end of context, with significantly less attention to the middle. When an engineering task requires sustained compliance with dozens of fine-grained rules — such as our HarmonyOS migration's semantic rules (29 in this configuration) + compiler restrictions (11) + execution disciplines (18) — these rules are scattered across multiple documents, repeatedly referenced, compacted, and reloaded across long sessions. Their effective binding force decays over time. Not deleted on paper, but **down-weighted in the AI's cognitive allocation.**

More subtly, AI develops "pattern inertia" — after the first few files pass smoothly, it reuses the same behavioral patterns and stops making file-specific distinctions for later, different files. This isn't rules being forgotten; it's **the binding between rules and specific contexts loosening.** The AI remembers the literal text of the rules but forgets which rule should be invoked for "this particular file."

---

## 3. Our Enhancement: A Four-Dimensional Systematic Defense

These four pain points don't operate independently — they interweave throughout the engineering process. Objective drift lowers standards; sub-agent neutrality loss lets degraded code pass audit; memory brittleness means the post-compaction AI doesn't even know standards dropped; attention entropy ensures remaining rules can't be effectively enforced.

So our solution isn't four standalone patches — it's a **unified control architecture spanning all phases and all sessions.**

---

### Solution 1: "Convergence-Driven Nested Loops + Mandatory Checkpoints" to Anchor Objectives

Since objective drift stems from the absence of external anchors, the solution is to **decompose a vague long-term goal into a sequence of short-term sub-goals with explicit convergence criteria**, with hard checkpoints at each node.

Our process contains 6 Phases with **6 mandatory checkpoints (CP0-CP5)** between them. Each CP includes automated verification items plus human confirmation. The AI must output CP PASS before entering the next Phase. CP1 verifies plan completeness, CP3 verifies compilation output, CP5 verifies runtime behavior — the objective is recalibrated at every phase boundary. "Unnoticed drift" is impossible because every entry condition is explicitly verified.

Within the Phases, we designed three nested convergent loops:

| Loop | Granularity | Convergence Condition | Hard Cap |
|------|------------|----------------------|:---:|
| **Loop 1 (Per-File Micro-Loop)** | Single file | Auditor PASS + Build 0 errors → file complete | 3 rounds; if not passed, deferred to Phase 4 audit |
| **Loop 2 (Audit-Fix-Reaudit, Dual-Nested)** | Project-level | Outer loop controls audit rounds (max 5); inner loop drives fix convergence until clean (no new Critical/High) | Outer 5 rounds; inner convergence-driven |
| **Loop 3 (Global Test Loop)** | Entry + runtime | Errors/failures monotonically decreasing + all PASS | Unlimited (but must be monotonic) |

Loop 3's "unlimited rounds but must monotonically decrease" is an intentionally designed mathematical constraint — it ensures every fix moves **forward**. If error counts plateau or rise, the system auto-pauses for human intervention. This fundamentally eliminates the pointless "fixing and re-fixing in the same place but never converging" drain.

---

### Solution 2: "Role-Separated Sessions + Information Isolation" to Rebuild Audit Independence

Since sub-agent neutrality is dissolved by the parent session's information field, the solution isn't "write a stricter Auditor prompt" — it's to **move the auditor entirely outside the author's information space.** Not switch agents — switch sessions.

This is the most critical architectural decision in the entire system:

- **Session 1 (Phase 1-3):** AI's role is **code author**. It loads 01-planner.md (the planning document). Its goal is to produce code that compiles. Session 1 **does NOT load 08-project-audit.md** — the audit document never enters the author's knowledge scope.

- **Session 1 is forcibly terminated after Phase 3 completion.** This is a hard rule in the runbook, not a "suggested pause." The AI outputs a structured Session Bridge prompt; the operator copies and pastes it into a **brand-new Session 2**.

- **Session 2 (Phase 4-6):** AI's role is **independent auditor**. It has **never written a single line of code. Never seen 01-planner.md.** It doesn't know "why it was designed this way," so it cannot be persuaded by design intent — it can only verify from the code itself and original library behavior, adversarially. It loads 08-project-audit.md (18 execution disciplines + 10 blind-spot checklist + two-stage deep audit framework), auditing Session 1's output with knowledge Session 1 never had access to.

The information asymmetry is intentionally constructed:

| | Session 1 (Author) | Session 2 (Auditor) |
|------|:---:|:---:|
| Knows "why it was designed this way" (01-planner.md) | ✅ | ❌ (sees only disk JSON) |
| Knows "what audits most commonly miss" (08-project-audit.md) | ❌ | ✅ |
| Has author attachment to the code (wrote these files) | ✅ | ❌ (never wrote a line) |
| Objective function | Get it to compile | Find every problem |
| Forcibly terminated after Phase 3 | ✅ (hard rule) | — |
| Loads 03-auditor.md (syntax auditing) | ✅ | ✅ |
| Loads 08-project-audit.md (global security/quality audit) | ❌ | ✅ |

This asymmetry mirrors the real-world relationship between construction crews and independent inspectors: **an inspector doesn't need to understand every design intent of the builder — they only need to check the output against the blueprints, item by item. If the builder's intent didn't materialize as correct code, that's the builder's problem.** Independent auditing isn't "nice to have" — our actual data shows the same code, same rules, independent auditing finds nearly twice as many issues, with severity a full tier higher.

This also partially answers the open question Anthropic left in *"Effective harnesses"*: *"Is a single general-purpose coding agent sufficient, or do we need specialized testing agents, QA agents?"* Our practice answers: **Not sufficient. In quality-sensitive engineering tasks, an independent audit role isn't optional — it's an architectural necessity. And independence isn't achieved through different prompts alone — it requires different sessions, different information sets, and different conversational biographies.**

Furthermore, we further subdivided the audit layer internally — **03-auditor.md** audits single-file syntax compliance ("is the code written correctly?"), while **08-project-audit.md** audits cross-file global security and completeness ("does the code do the right thing?"). These are complementary but non-substitutable — syntax compliance doesn't guarantee semantic correctness, and semantic correctness doesn't guarantee functional completeness. This layering also corresponds to AuditBench's finding — audit effectiveness depends on information asymmetry between auditor and audited. Same-layer auditing is more susceptible to self-rationalization than cross-layer auditing.

---

### Solution 3: "Gatekeeper + Embedded State Snapshots" to Counter Memory Brittleness

Memory brittleness is fundamentally about **project state residing in volatile media (the context window).** The conventional approach is to persist to files, but files require active read/write, and compaction is an asynchronous surprise event — you can't predict when it will strike.

Our approach: make state snapshots a natural component of the conversation flow, requiring no external maintenance.

The G1 file-gate output format is precisely designed to carry complete state information:

```
╔══════════════════════════════════════════════════════════╗
║  ⛔  G1 FILE GATE — HARD STOP                            ║
║                                                          ║
║  ✅ [filename] DONE — Awaiting Next | {X}/{Y} | Streak: {streak}/5 | Token: {used}K/{budget}K
╚══════════════════════════════════════════════════════════╝
```

This single line contains all variables needed for recovery: current file, progress, quality trend, token consumption. It's automatically output after every file completion, making it **naturally redundant** — even if compaction loses one or two G1 lines, the previous one still enables position recovery.

When compaction occurs, the recovery process is designed as three-tier escalation:

- **Tier 1 (Attempted auto-recovery)** — AI scans the conversation summary for G1/CP lines to locate the breakpoint
- **Tier 2 (Single-sentence human trigger — the standard path)** — Operator recognizes compaction, sends a fixed command, AI is forced to load the protocol text and execute recovery
- **Tier 3 (Direct human positioning — ultimate fallback)** — Operator directly tells AI the position; AI resumes from the specified breakpoint

The rationale for the three-tier design: **keep recovery initiative with the operator, but provide the operator with the minimal tool needed to execute recovery.**

This differs from the community pattern of "PreCompact hook → external file → SessionStart re-injection." Our approach doesn't depend on unstable hook APIs, adds no external dependencies, and works in any Claude Code environment.

---

### Solution 4: "Pattern-Driven Dynamic Intervention" to Counter Attention Entropy — Human Monitoring and Intervention at Key Nodes

Attention entropy is fundamentally about **AI's rule compliance decaying over time.** The conventional approach is "make rules more prominent" — shorter documents, bolder formatting, more frequent repetition. But these are passive measures with diminishing returns.

Our approach: **don't rely on the AI remembering rules; let the system detect rule-violation patterns and intervene actively.**

The G2 pattern detection continuously monitors 5 anomaly signals throughout Loop 1:

| Trigger | Detection Pattern | Intervention Logic |
|---------|------------------|-------------------|
| **T1** | Same violation ID appears in ≥2 consecutive files | Inject the corresponding rule patch directly into the Executor prompt — fundamentally altering the distribution of AI's subsequent outputs |
| **T2** | Single file passes after ≥2 rounds (struggle-then-pass) | No pause, but log — reviewed in batch at CP2 as a systemic weak signal |
| **T3** | streak ≥5 (5 consecutive files pass in 1 round) | Reverse intervention — AI may have entered a "comfort zone"; propose batch generation but require human decision |
| **T4** | Build errors from multiple files point to same dependency | Pause current file, fix dependency first — topology-aware scheduling |
| **T5** | 3 consecutive files each exhaust 3 rounds | **Emergency fuse** — this isn't "try harder," it's systemic failure requiring human diagnosis of prompt/strategy |

The elegance of this mechanism is that **it requires no metacognitive participation from Claude.** T1 detection is pure pattern matching (same violation ID appearing across different files' audit results); once triggered, intervention executes immediately (rule patch injected into the Executor prompt). The AI doesn't need to "realize it made the same mistake again" — the system makes that judgment on its behalf.

The G1 streak counter provides a quantified metric for per-file audit difficulty, allowing the operator to perceive global quality trends without reviewing every audit report file-by-file. The 10 blind-spot items in 08-project-audit.md (B1-B10) — "only auditing success paths, missing catch blocks," "trusting comments that say 'already handled,'" "marking pure computation functions as low-risk and skipping them" — are experiential patterns distilled from multiple real migrations, applied to compensate for AI's attention decay using human pattern recognition.

---

## 4. The Unified Philosophy Behind These Designs

These four solutions aren't isolated. They share several deep principles:

**1. The Principle of Distrust.** The entire system rests on one premise: AI output must be systematically distrusted — not maliciously, but as engineering "verification-driven" practice. Code must pass Auditor; Auditor must pass Session 2's independent audit; all output must pass CP checkpoints. This chain of distrust runs from architecture's top to bottom. This aligns with Petri's finding — auditing cannot rely on the audited party's conscientiousness; it must be externalized as independent, adversarial verification.

**2. The Principle of Asymmetry.** Verification should cost far less than generation, and the verifier's information set should differ from the generator's. A single G1 line contains complete state; 03 audits single files without needing global design knowledge; 08 audits globally without needing Phase 1 decision intent — each layer performs independent verification with less information.

**3. The Structural Position of Human-in-the-Loop.** The key to "human-AI collaboration" is where and how the human participates. In our system, human participation is precisely placed at three architectural nodes: G1 pacing decisions after each file, G2 intervention decisions upon trigger, and CP checkpoint confirmations. The shared characteristic of these three nodes: they are all **meta-verification that AI cannot self-execute** (verification of verification). One must remain clear-headed; the operator cannot afford to skip scrutiny at key nodes. Under long-context and compaction conditions, AI's deceptive tendencies are infinitely amplified.

**4. The Principle of Distributed State.** Project state must not reside in a single location; it requires multi-layer redundant backup with mutual corroboration. G1 lines (file-level snapshots in the conversation stream), CP lines (phase-level anchors in the conversation stream), 06-metrics.md (persistent on-disk records), and git tags (version markers independent of the AI conversation system) — four mutually redundant systems.

---

## 5. The Engineering Value of "Dynamic Workflow + Convergent Loops + Role-Separated Sessions + Gatekeeper"

**Boris Cherny**, Anthropic founder and the actual builder of Claude Code, stated publicly in a 2026 interview: *"I don't prompt Claude anymore. I have loops running that prompt Claude and figuring out what to do. My job is to write loops."*

Google's **Addy Osmani** subsequently formalized this paradigm as **Loop Engineering**, defining it as the fourth paradigm shift in AI-assisted programming: Prompt Engineering (crafting good single prompts) → Context Engineering (dynamically assembling the right context) → Harness Engineering (building runtime environments for agents) → **Loop Engineering (designing autonomous systems where humans stand outside the loop, defining rules and convergence criteria).**

We believe, however, that while the importance of prompts has relatively diminished, reliable prompts and markdown documents remain the key that starts the engine. The core of Loop Engineering isn't "eliminating prompts" — it's **transforming prompts from "single-use consumables" into "reusable engineering assets."** In our system, the 01-planner, 02-executor, 03-auditor, 08-project-audit, and other prompt documents are essentially solidified engineering methodology. They weren't rewritten for each migration — they were crafted once, verified repeatedly, and iteratively refined as "agent rule code." Loops are the skeleton, gatekeepers are the nervous system, and well-crafted prompts remain the flesh and blood. Prompts and loops are not in a substitution relationship — they are in a symbiotic relationship.

We open-source this methodology and practice not to provide a "one-click migration tool" — we provide a **reproducible engineering process architecture as a reference.** The same design patterns — convergent loops + role-separated sessions + gatekeepers + embedded state snapshots — can apply to any engineering task requiring AI reliability over long time horizons. HarmonyOS third-party library migration was our starting point and validation ground, but the methodology itself is universal.

If you, too, face the challenge of "letting AI independently complete a long engineering project" — whether code migration, automated testing, documentation generation, or continuous refactoring — we hope this system provides a reference starting point. Not "copy our prompts," but understand why we designed the architecture this way, then build your own loops and gates in your own domain.

---

**Further Reading:**

- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — Anthropic Engineering Blog, Justin Young, 2025.11
- [Petri: Parallel Exploration Tool for Risky Interactions](https://github.com/anthropics/petri) — Anthropic, 2025.10
- [LongHorizonDeception](https://arxiv.org/abs/2510.03999) — University of Wisconsin, 2025
- [AuditBench](https://arxiv.org/abs/2602.22755) — 2025
- [Lost in the Middle](https://arxiv.org/abs/2307.03172) — Liu et al., 2023
