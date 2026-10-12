<div align="center">

# 0day-skill

**Multi-Agent White-Box Code Auditing · Powered by AI Subagent Swarms**

English | [简体中文](README_CN.md)

[Quick Start](#quick-start) · [How It Works](#how-it-works) · [The Six-Question Gate](#the-six-question-gate) · [Example Finding](examples/finding-example.md)

</div>

> [!IMPORTANT]
> For **authorized security testing only** — penetration testing engagements, CTF competitions, auditing your own code, or defensive research. Users are solely responsible for ensuring their testing is authorized.

A portable **prompt-engineering asset** (pure Markdown, no executable code) that turns any AI coding client with "skills + subagents" support — [ZCode](https://z.ai), Claude Code, or any client compatible with the `.agents` directory layout — into a **multi-agent white-box code audit pipeline**.

**What it finds:** unauthenticated, weaponizable vulnerabilities — SQL injection, file upload leading to RCE, arbitrary file read/download, deserialization, command/expression/template injection, and SSRF — in any source tree or decompiled artifact (Java / PHP / Python / Go / Node / C#, including .NET DLLs, JARs, and firmware images).

---

## Highlights

**🕷️ Kill-chain hunting, not pattern matching**
- 🤖 **Commander + 4 subagent roles** — recon · sink-hunter · taint-tracer · verifier, each a fresh isolated context
- 🔁 **Backward taint tracing** — from every dangerous sink back to its source, three-state verdicts (CONFIRMED / BLOCKED / GAP)
- 🎯 **Six-Question Gate** — no finding is filed unless it is unauthenticated, works on default deployment, and survives without rotatable secrets
- 🧾 **Frontend trigger point required** — every finding carries the exact HTTP request and the UI element it fires from

**💾 State on disk, sessions as consumables**
- 🗂️ **Coverage matrix** (modules × 6 vulnerability classes) drives resumable, gap-free exploration
- ♻️ **One `/audit` per conversation** — swap to a fresh chat every round; progress lives in `audit-state/`, never in chat history
- 📉 **Measured anti-degradation** — new-session-per-round keeps context at 60–120K vs 190K+ degradation in long chats

**⚙️ Discipline encoded in prompts**
- 🛡️ **Adversarial verification** — verifier subagents independently re-judge tracer conclusions with line-number spot-checks
- 🚦 **Saturation signals** — the pipeline tells you when static analysis is exhausted instead of spinning forever
- 🔒 **Report gate** — reports are generated only on explicit user command, never mid-audit
- 🧩 **Combo chains don't count** — exploit chains reachable only through an already-counted root are filed as blast-radius evidence, not inflated findings

## How It Works

```
/audit  (one command — the rest is autonomous)
 │
 ├─ Recon subagent ──── fingerprint stack · map entry points · auth coverage · anonymous surface
 ├─ Sink-hunter ×N ──── per-module dangerous-API sweep → JSONL candidates
 ├─ Taint-tracer ────── backward source→sink tracing → CONFIRMED / BLOCKED / GAP
 ├─ Verifier ────────── six-question gate + frontend trigger point → findings/F-XXX.md
 │
 └─ Round ends: disk maintenance → saturation report → HALT (waits for user "continue")
```

**The core trick — state on disk, sessions as consumables:**

```
Session 1: /audit → runs one full round → writes audit-state/ → halts
Session 2: /audit → reads state.json → skips completed cells → continues
Session 3: /audit → next depth level (depth-1 → 2 → 3 → …N)
```

| Practice | Context peak | Judgment quality | Verdict |
|---|---|---|---|
| New conversation per round | ~60–120K tokens | Stable throughout | ✅ Recommended |
| Multiple rounds in one chat | 190K+ | Degrades late — hallucination risk rises | ❌ Avoid |

<details>
<summary><b>Why swapping conversations preserves quality</b></summary>

A single audit round accumulates skill-loading text + multiple subagent dispatch briefs + settlement summaries + state-file reads — context naturally climbs past 190K tokens, where instruction-following and detail retention measurably decay (observed: re-citing early conclusions, resurrecting already-BLOCKED chains, numbers drifting from disk). A fresh session restores judgment by re-reading only a trimmed `state.json` — **zero progress loss (everything is on disk), full quality reset**.

</details>

## The Six-Question Gate

Every candidate chain must survive six questions before it becomes a finding file:

| # | Question | What it filters |
|---|---|---|
| 1 | Reachable? | Route actually registered (annotation / config / convention) |
| 2 | Authenticated? | Filter coverage verified against the recon auth map, with `file:line` evidence |
| 3 | Exploitable? | Payload assembly, preconditions, prerequisites |
| 4 | **Unauthenticated** | No user/employee credentials anywhere in the chain |
| 5 | **Default deployment** | Works on vendor defaults — no admin misconfiguration required |
| 6 | **No rotatable secret** | Changing a hardcoded password must not kill the bug (code-defect roots exempt) |

Fail any of 4–6 → the chain is recorded in `chains.md` as a **chain asset** (combination material), never counted as a finding. **Combos don't count** — a chain that only becomes exploitable through an already-counted root documents that root's blast radius, not a new vulnerability.

## Quick Start

```bash
# 1. Clone
git clone https://github.com/xijiangyue-0op/0day-skill.git
cd 0day-skill

# 2. Install (copy to user-level directories)
cp -r zcode/skills/*  ~/.agents/skills/
cp -r zcode/agents/*  ~/.agents/audit/agents/
cp -r zcode/commands/ ~/.agents/commands/

# 3. Open the target source tree as your workspace, then type:
/audit
```

<details>
<summary><b>Other compatible clients</b></summary>

Any client with "markdown skills + a subagent tool" can run this: use `zcode/skills/audit-orchestrator/SKILL.md` as the commander system prompt and `zcode/agents/*.md` as self-contained task briefs injected into your subagent dispatcher.

</details>

### Recommended Environment

| Choice | Client | Model |
|---|---|---|
| **Primary** | [ZCode](https://z.ai) | GLM-5.3 |
| Alternative | [DeepSeek Harness](https://www.deepseek.com) | deepseek-v4.1-flash |

Both share the same skill system (single-source role cards); methodology and judgment criteria are identical — only the dispatch layer adapts to rate limits.

## Vulnerability Classes

<details>
<summary><b>The Six (click to expand)</b></summary>

| ID | Class | ID | Class |
|---|---|---|---|
| sqli | Unauthenticated SQL injection | deser | Deserialization (all flavors) |
| upload | File upload → RCE | rce | Command / code / expression / template (SSTI) injection |
| fileread | Arbitrary file download / read / traversal | ssrf | Server-side request forgery |

Blacklisted (never filed): XSS, CSRF, CRLF, clickjacking, pure info-leak, weak passwords, anything low-impact or high-interaction.

</details>

## Output Artifacts

```
audit-state/
├── state.json        # Coverage matrix + index — the single source of truth for resume
├── recon.md          # Stack fingerprint · entry-point map · auth coverage · anonymous surface
├── findings/F-XXX.md # Confirmed kill-chain dossiers (taint chain + trigger + six-question table)
├── chains.md         # Chain assets (conditional/combination chains, not counted)
├── pending.md        # Live backlog: gaps · second-order leads · next-round queue
└── log.md            # Full timeline: cells closed · corrections · round records

audit-reports/Audit-Report-<system>-<date>.md   # Generated ONLY on explicit user command
```

See [examples/finding-example.md](examples/finding-example.md) for a real finding dossier — taint chain, trigger request, six-question table, and the exact frontend entry point.

<details>
<summary><b>Built-in discipline (already encoded in the prompts)</b></summary>

- Every finding must carry a frontend trigger point;
- Truncation forces follow-up cells — a note without a matrix cell is a lost lead;
- Merged families need ≥10% spot-checks + line-number verification;
- Indirect chains (reflection / config-driven) are dead ends only if proven, not assumed;
- Round caps (step ≥150 or 90 minutes → must close the round); reads >30KB must be trimmed;
- The commander never does a subagent's job (failure → redispatch once → back to pending);
- Report gate: never generates a report during audit — only on explicit user command.

</details>

## Project Layout

```
0day-skill/
├── zcode/                     # ZCode package (orchestrator skill + 5 role cards + 3 commands)
│   ├── skills/audit-orchestrator/SKILL.md
│   ├── agents/{recon,sink-hunter,taint-tracer,verifier,report-writer}-agent.md
│   └── commands/{audit,audit-status,audit-report}.md
└── examples/
    └── finding-example.md     # findings/F-XXX.md dossier format
```

## License

[MIT](LICENSE) — for authorized security testing only.

## ⚠️ Disclaimer

This project is a **prompt-engineering asset** (pure Markdown text) and contains no executable code. Users are solely responsible for ensuring their testing is authorized and for all consequences of use. The author is not responsible for any misuse.

---

## Community

<div align="center">

**代码审计 skill 交流群 · Code Audit Skill Discussion Group**

Questions, ideas, or field reports? Join the group chat.

<img width="450" height="680" alt="Code Audit Skill Discussion Group QR code" src="https://github.com/user-attachments/assets/60bea1d5-be2e-4234-857e-9d4396733595" />

</div>
