---
name: agent-audit
description: >
  Token-economy/clarity audit of .claude/agents/*.md sub-agent prompt files —
  a specific target or the whole roster. Coordinates prompt-auditor
  (read-only findings) with the orchestrator's own apply → live self-test →
  confirm loop. Never runs on its own; explicit request only, not part of
  any autonomous tail.
  Use when: "оптимизируй агентов", "почисти промпт агента X", "agent-audit",
  "audit .claude/agents", "проверь промпты на дубли", "раз в месяц оптимизация агентов".
---

# /agent-audit

Entry point for a token-economy/dedup pass over sub-agent prompt files.
Coordinates `prompt-auditor` (finds + proposes) with the orchestrator's own
apply/self-test/confirm loop (`prompt-auditor` never edits anything itself —
sub-agents don't spawn sub-agents, so the live self-test step cannot live
inside it).

## Usage

```
/agent-audit qa-impact-agent                 # one target, by agent name or path
/agent-audit qa-impact-agent qa-logs-agent   # explicit list
/agent-audit all                             # whole .claude/agents/*.md roster
```

## Algorithm

### 1. Resolve targets

- A bare name (`my-agent`) → `.claude/agents/my-agent.md`.
- `all` → every `.claude/agents/*.md` file in your roster. If any agent is
  owned by someone else or otherwise out of scope for this kind of pass,
  maintain your own exclusion list here (adapt this bullet to name them) —
  never audit an agent the user has flagged as off-limits.
- For each resolved target, check whether a paired `.claude/skills/*/SKILL.md`
  exists (by grepping skill descriptions for the agent's name/subagent_type —
  most agents have one, some don't: an agent invoked only inline by the
  orchestrator has no dedicated skill file). Pass the pairing to the auditor
  when found.
- No target resolvable → `BLOCKED_NO_TARGET`.

`all` is a heavy multi-file pass (potentially 20+ files) — confirm with the
user before launching it: "Полный ростер — N файлов. Погнали по всем, или
начнём с нескольких?"

### 2. Launch `prompt-auditor` per target

One call per target file (independent files → launch in parallel). Pass the
absolute path(s) and the paired SKILL.md path, if any:

```python
Agent(subagent_type="prompt-auditor", prompt="""
Audit this sub-agent prompt file for token-economy/clarity/structural issues.

Target: <absolute path to .claude/agents/<name>.md>
Paired skill (if any): <absolute path to .claude/skills/<skill>/SKILL.md, or "none">

Follow your full process: internal duplication, dead/stale references
(verify against current repo state), drifted twin-lists, compressible prose
with no incident citation, structural placement of high-stakes rules, and
latent contradictions between rules. Classify every candidate per your
CRITICAL RULES. Produce the Output Contract report.
""")
```

### 3. Orchestrator memory cross-check

For every finding the auditor flagged **"no repo documentation found
justifying this as intentional"** — before accepting it as safe, check this
session's own memory index (if one exists) for the target agent's name or the
specific pattern, the same way you'd check whether a recurring section (e.g.
a "Hard Rules" recap block) was ever called out elsewhere as a deliberate
recency reinforcement rather than accidental duplication. A finding memory
*does* flag as intentional is dropped from the plan, not applied.

### 4. Self-test before presenting anything (not after)

Apply, verify, then show the plan — never the other way around:

1. **Apply** the surviving candidate edits to disk (the orchestrator edits
   directly — these are prompt/doc files, not product code, so no separate
   writer sub-agent is needed).
2. **Self-test, scaled to what's actually feasible per agent:**
   - If the target agent has a cheap, safe **live** invocation available (a
     read-only external API it calls) — run it on a real historical case
     with the *unedited* prompt (if not already known from a prior round)
     and the *edited* prompt, diff the two outputs line-by-line for
     behavioral equivalence (same verdict/risk level/label discipline/output
     contract).
   - If no such cheap live target exists (e.g. the agent needs a real
     product-repo diff context, or a running app) — do a **structural
     dry-run** instead: confirm every constraint the edit removed still
     exists somewhere else in the file (or was never removed, only
     reworded), confirm no orphaned cross-reference was introduced. Say
     explicitly in the plan which kind of verification each file got — never
     silently skip verification.
3. **Any test showing even minor degradation** (weaker labeling, dropped
   constraint, different structure) → drop that specific finding, don't ship
   it; keep the rest.
4. **Revert everything to the pre-audit state** before presenting the plan —
   nothing stays applied until the user confirms.

### 5. Present the plan

Per file: baseline size, each surviving finding with its exact before/after
diff and rationale, self-test method used and result, total estimated
savings (lines/chars). Explicitly confirm: no CRITICAL-RULES-equivalent
content, Output Format, or Fail-fast Sentinels were touched; nothing shipped
without a passing self-test (or an honest note that only a structural check
was possible).

Stop and wait for explicit confirmation — never apply past this point
without it.

### 6. On confirmation

Re-apply the confirmed edits (re-verify targets are still at the reverted
baseline first, same re-check discipline as step 4). No version bump for
this kind of pass unless it happens to coincide with an actual release. If
your project maintains a separate agent/skill roster doc, and the audit
renamed, rescoped, or removed anything it describes, update those rows too.

## Rules

- Never let `prompt-auditor` edit anything — it has no Write/Edit tools by
  design; if a report includes a diff, the orchestrator applies it, not the agent.
- Never skip the self-test step, even for `all` mode — scale its rigor
  (live A/B vs structural dry-run) to what's feasible per file, but always
  do one and say which.
- Never apply anything before presenting the plan and getting confirmation.
- Any agent on your own exclusion list is always out of scope, in every mode
  including `all`.
- One `/agent-audit` call = one audit round (of however many target files
  were resolved) — not a standing watch job. This skill runs only on
  explicit request, never automatically.

## Output Contract

A consolidated plan covering every target file's findings, self-test
results, and total savings — relayed directly to the user, not summarized away.

## Fail-fast

```
BLOCKED_NO_TARGET
No agent name/path given and "all" not specified
Needed from user: an agent name, a list of names, or "all"
```
