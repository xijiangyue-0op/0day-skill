# 0day-skill

English | [简体中文](README_CN.md)

Multi-agent white-box code auditing with AI. One command — `/audit` — automatically dispatches subagents to recon, hunt sinks, trace taint chains, and verify findings into submittable vulnerability dossiers.

> ⚠️ For authorized security testing only.

## Install & Run

```bash
git clone https://github.com/xijiangyue-0op/0day-skill.git
cp -r 0day-skill/zcode/skills/*  ~/.agents/skills/
cp -r 0day-skill/zcode/agents/*  ~/.agents/audit/agents/
cp -r 0day-skill/zcode/commands/ ~/.agents/commands/
```

Open the target source directory and type `/audit`.

Recommended: ZCode + GLM-5.3 (alternative: DeepSeek Harness + deepseek-v4.1-flash).

## What It Finds

SQL injection / file upload RCE / arbitrary file read / deserialization / RCE / SSRF — **unauthenticated** ones only, each with its trigger request.

## How to Use

```
New chat: /audit → runs one full round → halts
Close it, open a new chat: /audit → resumes from where it left off
Repeat until it says "audit complete"
```

**Always start a fresh chat per round** — progress lives on disk, long chats degrade the model.

When done, say "write the report" to generate `audit-reports/Audit-Report.md`.

## Output

```
audit-state/findings/F-XXX.md   # Vulnerability dossiers (taint chain + trigger + six-question table)
audit-state/chains.md           # Chains missing one link (not counted)
audit-reports/Audit-Report.md   # Final report
```

Dossier format example: [examples/finding-example.md](examples/finding-example.md)

## Community

<img width="300" alt="Code Audit Skill Discussion Group" src="https://github.com/user-attachments/assets/60bea1d5-be2e-4234-857e-9d4396733595" />

## License

MIT — authorized testing only.
