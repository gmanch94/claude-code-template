# CLAUDE.md

Guidance for Claude Code working in this repo. Auto-loaded at the start of every session.

---

## What this repo is

**[PROJECT NAME]** — [one sentence: what it is, who it's for, what problem it solves].

[Optional: one sentence on what this repo is NOT — helps Claude avoid scope creep.]

<!-- Example: **Retail Pricing Engine** — ML service that updates product prices daily using demand signals; serves the merchandising team. Not a general pricing framework — single-tenant, single-SKU-hierarchy only. -->

---

## Session-start protocol

Before any tool calls beyond basic orientation:

1. Read [`NEXT_SESSION.md`](NEXT_SESSION.md) — resume bookmark: HEAD, branch, what landed last session, open items, things NOT to do without explicit instruction
2. Read [`LESSONS_LEARNED.md`](LESSONS_LEARNED.md) — process lessons from prior sessions; re-reading prevents repeat mistakes
3. Read this file — repo posture and constraints
4. `git status` + `git log --oneline -5` — confirm state matches the bookmark
5. Only then ask the user what they want to work on — do not start anything proactively

---

## Source of truth

[Optional — delete this section if your project has no single canonical reference file.]

**[FILENAME]** is the canonical source for [what it defines — e.g., schema, feature list, API spec, pricing].

The `[Used in / Referenced by]` column (or equivalent) maps each row to downstream files that need follow-up edits.

**Never** edit a derived file without updating the source of truth in the same change.

---

## Repo structure

```
[PROJECT NAME]/
├── CLAUDE.md                  ← This file
├── LESSONS_LEARNED.md         ← Process lessons (re-read each session)
├── scratch/                   ← Gitignored; NEXT_SESSION.md and personal workspace
├── context/                   ← Claude's persistent project memory
│   └── MEMORY.md              ← Memory index
├── prompts/                   ← System prompt templates (RAG, agent, chat, classifier, etc.)
│   └── README.md              ← Template index + how-to-use guide
├── templates/
│   ├── adr/ADR-TEMPLATE.md    ← ADR template (used by /adr skill)
│   └── skill/                 ← Skill authoring guide
│       ├── SKILL-TEMPLATE.md  ← Annotated template — copy to .claude/skills/<name>/SKILL.md
│       └── REFERENCE-TEMPLATE.md ← Optional reference doc template
└── [your project files here]
```

---

## Sprint workflow

For any non-trivial task, follow this order:

1. **Assumptions** — run `/office-hours` to surface unstated assumptions and produce a design doc before writing code
2. **Plan** — agree on approach; use `/tradeoff` if options need evaluating, `/adr` if a decision needs recording
3. **Implement** — build against the design doc; invoke Confusion Protocol if new ambiguity surfaces
4. **Review** — run `/review` before opening a PR
5. **Ship** — feature branch + PR; no direct commits to master
6. **Retro** — run `/retro` at end of session or sprint; write new lessons to LESSONS_LEARNED.md

Skip steps only with explicit agreement — not because the task feels small.

---

## Working conventions

[Fill in: start with 3–5 entries. Remove examples you don't use. Add more as conventions crystallize across sessions.]

- [Naming conventions — files, variables, branches]
- [How tests are run and what "passing" means for this project]
- [PR / commit conventions — branch naming, squash vs. merge, etc.]
- [Any file that must never be edited directly (generated files, lockfiles, etc.)]
- [Stack-specific: language version, formatter, linter commands]
- **Skill authoring (standing):** whenever a new skill is added under `.claude/skills/`, ALWAYS also (1) add a corresponding prompt template in `prompts/<name>.md` and (2) add the skill's row to the `docs/AUTOMATION.md` catalog (and the `prompts/README.md` index). A skill is not "done" until both exist. **Exempt from the prompt-template requirement:** workflow / facilitator / agent-spawning / operator skills that *are* the prompt or spawn a subagent rather than parameterize an LLM system prompt — `adr`, `office-hours`, `retro`, `review`, `tradeoff`, `prompt-review`, `api-audit`, `security-audit`, `security-model-init`, `doc-ci-check`, `rollback-checkpoint`. These still get a `docs/AUTOMATION.md` catalog entry but no `prompts/<name>.md`. A stale-check should not re-flag them. The same list is mirrored in `.claude/skills/doc-ci-check/SKILL.md` and `.github/workflows/doc-ci.yml` — keep all three in sync.

**Confusion Protocol** — when facing an architectural decision or ambiguous requirement, stop and surface the assumption explicitly before proceeding. Never guess on design decisions. Ask one targeted question instead of producing output that may be wrong.

**AGENTS.md interop (multi-host repos)** — if this repo will be used with Codex / Cursor / Gemini CLI / Aider alongside Claude Code, keep the project rules in a single source-of-truth `AGENTS.md` at the repo root and have `CLAUDE.md` IMPORT it via `@AGENTS.md` (or a symlink). Claude Code as of mid-2026 does NOT read `AGENTS.md` directly, so the import is required; the other tools read `AGENTS.md` natively. This pattern keeps a single rules file usable across hosts without duplication. If the template is Claude-Code-only, ignore this — `CLAUDE.md` alone suffices.

---

## Tone and output constraints

[Fill in: keep only what applies; delete the rest. These become hard constraints Claude follows without being reminded.]

- No emojis in [artifacts / commits / output] unless explicitly requested
- Numeric where possible — no adjectives doing numeric work ("significant improvement" fails; "42% latency reduction" passes)
- Every recommendation names a failure mode — no universally-best options
- Comments in code: only when the WHY is non-obvious. No "this function does X" comments.
- [Language / framework style guide link if applicable]

---

## Security primer

When this project grows a user-facing surface (auth, DB, API, file uploads, public reads):

1. **Commit #2 of that growth:** run `/security-model-init` to scaffold `docs/SECURITY_MODEL.md`. Don't write the next code change until §4's enforcement table has no empty cells (or the empty cells are tracked in §6 with target close dates).
2. **Before any multi-PR sprint touching DB/auth:** run `/security-audit`. Triage CRITICAL/HIGH must be fixed before sprint starts.
3. **Before production deploy:** run `/security-audit` again. Pre-launch gating, not post-launch reactive.

Universal mental model — **the API endpoint you wrote is one path to the data; auto-generated REST/GraphQL endpoints, mobile SDK queries, public file URLs are others.** Every invariant the API enforces must independently exist at the data layer (RLS / Firestore Rules / Hasura permissions / DB triggers / column-level REVOKE / service-role-only writes). "I check it in the action" is decorative if `curl` against the auto-surface bypasses it.

Full protocol: see `operating-philosophy.md` § Security thinking. Pre-merge independent reviewer policy applies to every PR touching auth flow, RLS / authorization rules, DB triggers/functions, payment state, or any "safety property" comment.

---

## Things to avoid

- Don't commit directly to `master` — all changes via feature branch + PR
- Don't push to remote without explicit user instruction
- Don't use long PowerShell here-strings for commit messages — hits 948-byte parse limit; use inline `-m "..."` instead

**Four failure modes to guard against (Karpathy):**
- **Wrong assumptions** — don't guess at intent; surface the assumption and ask
- **Overcomplexity** — don't add abstraction, generalization, or flexibility the task doesn't require
- **Orthogonal edits** — don't touch code outside the stated task scope; no drive-by cleanup
- **Imperative over declarative** — prefer describing the desired outcome over prescribing steps

---

## Automation

The full annotated catalog of skills, hooks, permissions, and scheduled routines lives in
**[docs/AUTOMATION.md](docs/AUTOMATION.md)**. It is a human reference: the Claude Code harness
lists every available skill (name + description) at session start, so you do not need the catalog
loaded to route to a skill. Quick pointers — skills in `.claude/skills/`, hooks in `.claude/hooks/`
(see [`.claude/hooks/README.md`](.claude/hooks/README.md)), permissions in `.claude/settings.json`.

New skills: add the row to [docs/AUTOMATION.md](docs/AUTOMATION.md) (not this file) plus the
`prompts/README.md` and `README.md` indices — doc-ci checks parity against `docs/AUTOMATION.md`.
