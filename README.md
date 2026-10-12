# 0day-skill — Multi-Agent White-Box Code Audit Framework

**English** | [简体中文](README_CN.md)

A portable prompt-engineering asset that turns any AI coding client with "skills + subagents" support (ZCode, Claude Code, or any client compatible with the `.agents` directory layout) into a **multi-agent white-box code audit pipeline** — a commander agent dispatches four specialized subagents (recon / sink-hunting / taint-tracing / verification), with all state persisted to disk for crash-proof resume and unlimited-depth exploration.

> What it finds: **unauthenticated, weaponizable vulnerabilities** — SQL injection, file upload leading to RCE, arbitrary file read/download, deserialization, RCE (command/expression/template injection), and SSRF — in any source code or decompiled artifact (Java/PHP/Python/Go/Node/C#, including .NET DLLs, JARs, and firmware images).

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

Every finding must survive the **Six-Question Gate** before it becomes a finding file:

1. Is it reachable? (route registered)
2. Is it authenticated? (filter coverage, with evidence)
3. Is it exploitable? (payload assembly)
4. **Unauthenticated** — no user credentials anywhere in the chain
5. **Works on default deployment** — no admin misconfiguration required
6. **No rotatable-secret dependency** — changing a hardcoded password must not kill the bug (code-defect roots exempt)

Chains that fail any of 4–6 are recorded as chain assets (combination material), not counted as findings. **Combos don't count** — a chain that only becomes exploitable through an already-counted root is evidence of that root's blast radius, not a new vulnerability.

## Recommended Environment

| Choice | Client | Model |
|---|---|---|
| **Primary** | [ZCode](https://z.ai) | GLM-5.3 |
| Alternative | [DeepSeek Harness](https://www.deepseek.com) | deepseek-v4.1-flash |

Both share the same skill system (single-source role cards); the audit methodology and judgment criteria are identical — only the dispatch layer adapts to rate limits.

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

### Other compatible clients

Any client with "markdown skills + a subagent tool" can run this: use `zcode/skills/audit-orchestrator/SKILL.md` as the commander system prompt and `zcode/agents/*.md` as self-contained task briefs injected into your subagent dispatcher.

## Vulnerability Classes (the Six)

| ID | Class | ID | Class |
|---|---|---|---|
| sqli | Unauthenticated SQL injection | deser | Deserialization (all flavors) |
| upload | File upload → RCE | rce | Command / code / expression / template (SSTI) injection |
| fileread | Arbitrary file download / read / traversal | ssrf | Server-side request forgery |

Blacklisted (never filed): XSS, CSRF, CRLF, clickjacking, pure info-leak, weak passwords, anything low-impact or high-interaction.

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

Every finding includes a **frontend trigger point** (HTTP request + where in the UI it fires from) — because a vulnerability you can't trigger is a hypothesis, not a finding.

## Built-in Discipline (already encoded in the prompts)

- Every finding must carry a frontend trigger point;
- Truncation forces follow-up cells — a note without a matrix cell is a lost lead;
- Merged families need ≥10% spot-checks + line-number verification;
- Indirect chains (reflection / config-driven) are dead ends only if proven, not assumed;
- Round caps (step ≥150 or 90 minutes → must close the round); reads >30KB must be trimmed;
- The commander never does a subagent's job (failure → redispatch once → back to pending);
- Report gate: never generates a report during audit — only on explicit user command.

## Repository Layout

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

[MIT](LICENSE) — for authorized security testing only (penetration testing engagements, CTF competitions, auditing your own code, defensive research).

## ⚠️ Disclaimer

This project is a prompt-engineering asset (pure Markdown text) and contains no executable code. Users are solely responsible for ensuring their testing is authorized and for the consequences of use. The author is not responsible for any misuse.

---

## Community

<div align="center">

### 代码审计 skill 交流群 / Code Audit Skill Discussion Group

Questions, ideas, or field reports? Join the group chat.

<img width="450" height="680" alt="Code Audit Skill Discussion Group QR code" src="https://github.com/user-attachments/assets/60bea1d5-be2e-4234-857e-9d4396733595" />

</div>
