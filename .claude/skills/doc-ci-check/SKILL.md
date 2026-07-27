---
name: doc-ci-check
description: Doc-CI gate for repos whose deliverable includes a document set. Scans for (1) count drift across README/CLAUDE.md/NEXT_SESSION.md/prompts indices, (2) broken relative links, (3) skill↔prompt↔docs/AUTOMATION.md↔README↔prompts/README parity, (4) ungraded external facts, (5) markdown table pipe-integrity (stray-pipe phantom tables). Severity-grouped report. Use BEFORE shipping any docs commit, AFTER adding a skill/prompt/hook, and as a pre-commit gate. Pair with `.github/workflows/doc-ci.yml` for CI enforcement.
---

# /doc-ci-check — Doc-CI Gate

Encodes the manual stale-check ritual into a single command. Pre-ship gate that prevents the staleness this skill was created in response to (NEXT_SESSION HEAD drift, "66 prompts" → 137, "Three hooks" → 13, 21 skills missing from README).

## When to invoke

- **BEFORE any commit** touching `README.md` / `CLAUDE.md` / `NEXT_SESSION.md` / `prompts/README.md` / `.claude/hooks/README.md` / `stacks/README.md`
- **AFTER adding a skill** (verifies all 5 artifacts landed: skill + prompt + AUTOMATION row + README row + prompts/README row — except for facilitator/workflow skills on the exempt list)
- **AFTER adding a hook** (verifies inventory tables, framing claims, and counts updated everywhere)
- **WEEKLY** as a routine via `/schedule` if the repo's deliverable is docs/skills
- **BEFORE merging a docs PR** as the content equivalent of a green test suite

## Inputs

None. Operates on the current working tree.

## Steps

1. **Run the deterministic checks** (shell — directly, not via subagent):

   a. **Count drift.** For each pair (declared count, actual count), flag mismatches:
      - `ls .claude/skills/ | wc -l` vs every aggregate count in README/CLAUDE/NEXT_SESSION/prompts-README/hooks-README
      - `ls prompts/*.md | grep -v README | wc -l` vs same files
      - `ls .claude/hooks/*.py | wc -l` vs same files
      - `ls decisions/ | wc -l` (if dir exists) vs NEXT_SESSION + README

   b. **Relative-link validity.** For each `[text](path)` in root-level `*.md`, every `path` must resolve. Skip anchors-only (`#foo`). Skip absolute URLs.

   c. **Skill ↔ artifact parity** for each skill in `.claude/skills/`:
      - Has `SKILL.md`? (required)
      - Has matching `prompts/<name>.md`? (required UNLESS in exempt list — see below)
      - Mentioned in `docs/AUTOMATION.md`? (required)
      - Mentioned in `README.md`'s slash-command tables? (required UNLESS exempt — same list)
      - Listed in `prompts/README.md` index? (required UNLESS exempt)

   d. **Reverse parity** for each prompt in `prompts/*.md` that matches a skill name: the corresponding `.claude/skills/<name>/` MUST exist. Standalone runtime prompts (chat-assistant, classifier, extractor, summarizer, …) are explicitly allowed — they have no skill counterpart by design.

   e. **Ungraded-fact scan.** Surface external claims (years, percentages, dollar amounts, version numbers, model IDs) in root `*.md` files that carry no source grade (`[H]`/`[M]`/`[?]`) and look like assertions of fact. Warn-only.

   f. **NEXT_SESSION.md staleness.** Parse the documented `HEAD = <hash>`; compare to `git rev-parse HEAD`. If different, flag CRITICAL.

   g. **Markdown table pipe-integrity.** For every real GFM table (header + `|---|` delimiter, outside code fences), each row's unescaped-pipe cell count must equal the delimiter's. A row with MORE cells means an unescaped `|` inside a cell (code span, math abs-value like `|y − ŷ|`, type union like `dict | None`) is shredding the table under GFM — flag CRITICAL, fix by escaping as `\|`. A row with FEWER cells (missing value; GFM pads a blank) is LOW/warn. Enforced in CI by `scripts/check_md_tables.py`.

2. **Group findings by severity:**
   - **CRITICAL** — broken link, NEXT_SESSION HEAD mismatch, skill with no SKILL.md, missing required artifact
   - **HIGH** — count drift (off by 2x or more), reverse-parity violation
   - **MEDIUM** — count drift (off by <2x), skill mentioned in docs/AUTOMATION.md but missing from README slash-command table
   - **LOW** — ungraded external fact, missing optional REFERENCE.md when SKILL.md exceeds 500 lines

3. **Produce the report** (see Output section).

4. **If findings exist, propose fixes** — do NOT auto-apply unless the user confirms. Always offer the auto-fix recipe for trivial cases (e.g., "I can update the count in 3 files").

5. **If clean,** print `OK — N skills, M prompts, P hooks, all parity holds.`

## Exempt list (skills with no prompt template by design)

These skills are workflow / facilitator / agent-spawning — they ARE the prompt or spawn a subagent rather than parameterize an LLM system prompt. They get an index entry in docs/AUTOMATION.md only (no prompt file, no prompts/README index row):

```
adr  api-audit  doc-ci-check  office-hours  prompt-review  retro  review  rollback-checkpoint  security-audit  security-model-init  tradeoff
```

Keep this list in sync with the `CLAUDE.md` "Skill authoring (standing)" rule and the GH Action below.

## Output

```
### Doc-CI Report — <repo>

Summary: <N> CRITICAL, <M> HIGH, <P> MEDIUM, <Q> LOW

CRITICAL
1. NEXT_SESSION.md HEAD mismatch — declared <hash>, actual <hash> (off by N commits)
2. README.md L42: broken link [text](path/that/does/not/exist)
...

HIGH
1. README.md L38: "66 system prompt templates" — actual count 137 (2.1× off)
...

MEDIUM
1. CLAUDE.md mentions /foo-bar but README.md slash-command tables do not include it
...

LOW
1. README.md L100: "GPT-4 costs $0.03/1K tokens" — ungraded external fact, no [H]/[M]/[?]
...

Fix recipe (if findings):
- A: Update count in README.md:38, NEXT_SESSION.md:55, prompts/README.md:?
- B: Move /edge-ml-deployment from OT section to Model deployment section
- ...
```

## Quality bar

- Always run all seven checks — never short-circuit even if the first one passes
- Always group by severity — never present findings as a flat list
- Never auto-fix without user confirmation; always show the fix recipe first
- Never flag a skill in the exempt list as missing-prompt
- Always treat NEXT_SESSION HEAD mismatch as CRITICAL — it cascades into every other check being against stale state
- If the repo has no `.github/workflows/doc-ci.yml`, recommend adding it so the same checks run in CI

## What this skill does NOT do

- Does NOT rewrite docs autonomously — only flags and proposes
- Does NOT run linters/formatters on prose — out of scope
- Does NOT replace `/security-audit` or `/code-review` — different category
- Does NOT enforce branding/voice — for that see `~/.claude/rules/anti-ai-voice.md`

## Companion CI workflow

See `.github/workflows/doc-ci.yml` for the GH-Action version of checks (a)–(d) + (g) — the last via `scripts/check_md_tables.py`. The skill and the workflow share the same exempt list and the same count scans. When the rules change, update both.
