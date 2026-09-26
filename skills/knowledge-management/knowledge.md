# Skill: Knowledge Management

Capture, structure, and retrieve durable knowledge about a codebase so that every agent — regardless of role — can benefit from what other agents have already learned.

---

## Why This Skill Exists

Every agent starts a task with zero context about the target repository. Without a shared memory:

- The same discovery work (entry points, build commands, data flows, gotchas) is repeated by every agent.
- Tribal knowledge discovered by one agent is lost when that session ends.
- Agents contradict each other because each derives its own "truth" about the code.

This skill defines a **single shared knowledge base** in the target repository that all agents write to and read from.

---

## The Knowledge Lifecycle

```
Observe → Capture → Structure → Index → Verify → Retrieve → Refresh
```

1. **Observe** — during any task you read, run, or change code.
2. **Capture** — anything *durable* you learned goes into the knowledge base, not just into your context.
3. **Structure** — write it into the correct file using the standard entry format.
4. **Index** — register it in `docs/knowledge/INDEX.md` so others can find it.
5. **Verify** — every entry must cite evidence (file path, command output, or test result).
6. **Retrieve** — before investigating a repository, read the knowledge base first.
7. **Refresh** — update or mark stale entries whenever the underlying code changes.

---

## Where Knowledge Lives

Store the knowledge base in the **target repository**, not in the agent session:

```
docs/knowledge/
├── INDEX.md                 # Entry point: catalogue, freshness, ownership
├── overview.md              # Purpose, stack, how to build/run/test
├── architecture.md          # Components, boundaries, data flows, diagrams
├── conventions.md           # Naming, style, and patterns actually in use
├── operations.md            # Deploy, debug, monitor, rollback procedures
├── gotchas.md               # Sharp edges, hidden invariants, landmines
├── modules/                 # One deep-dive per bounded area
│   └── <area>.md
├── decisions/               # Architecture/decision records
│   └── ADR-NNN-<slug>.md
└── changelog.md             # Dated log of every knowledge update
```

If the repository already has a documentation home (e.g. `docs/`, `wiki/`, `confluence/`), integrate there instead of creating a parallel tree — **one source of truth only**.

---

## Entry Format

Every knowledge item follows the same shape so any agent can parse it:

```markdown
### <Short fact in one line>

- **Type**: fact | decision | convention | gotcha | how-to | command
- **Area**: <module or subsystem>
- **Evidence**: `path/to/file.ext:12`, test run, or command output
- **Confidence**: high | medium | low
- **Added**: 2026-09-26 by <agent/role>
- **Verified**: 2026-09-26

<1–5 lines of explanation. State the "why", not just the "what".>

See also: `docs/knowledge/modules/<area>.md`
```

### Quality bar for an entry

- [ ] Durable — will still be true next month without code changes.
- [ ] Non-obvious — not something you can read off a single file header.
- [ ] Evidenced — cites a path (`file:line`), a command, or an observed result.
- [ ] Scoped — belongs in one area; cross-link instead of duplicating.
- [ ] Secrets-free — no tokens, credentials, internal hostnames, or private data.
- [ ] Actionable — tells the next agent what to do or avoid.

### Do NOT capture

- Ephemeral state (current branch, running processes, one-off error messages).
- Facts the code states explicitly and unambiguously at the cited location.
- Speculation presented as fact (mark confidence `low` and label as hypothesis).
- Anything you have not verified.

---

## Retrieval Protocol (read before you investigate)

Follow this order at the start of any task in an unfamiliar repository:

1. **Check for a knowledge base** — look for `docs/knowledge/INDEX.md` (or an equivalent documented location).
2. **Read `INDEX.md`** — get the map: what exists, how fresh it is, what is marked stale.
3. **Read the file most relevant to your task** — `overview.md` for orientation, `modules/<area>.md` for depth, `gotchas.md` before touching risky code.
4. **Check freshness** — every entry carries `Verified:`; treat anything older than the last relevant commit as suspect.
5. **Verify before acting** — open the cited `file:line` and confirm the entry still matches the code.
6. **Cite while working** — reference knowledge entries in your output (`docs/knowledge/architecture.md`) so others can trace your reasoning.
7. **Report gaps** — if the knowledge base is missing or wrong for what you need, say so explicitly and capture the correction.

```
Knowledge base present?
├── yes → INDEX → relevant file → verify → act → update knowledge
└── no  → investigate → capture findings → create INDEX → act
```

---

## Capture Protocol (write when you learn)

Capture at these moments:

| Moment | What to capture |
|---|---|
| Finished onboarding a repo | `overview.md`, `architecture.md`, initial `INDEX.md` |
| Discovered a build/test/run command that works | `overview.md` → Commands section |
| Traced a request/data flow end-to-end | `architecture.md` or the relevant `modules/` file |
| Learned an unwritten convention by reading code | `conventions.md` |
| Hit a non-obvious failure or trap | `gotchas.md` |
| Made or learned of a significant design decision | `decisions/ADR-NNN-*.md` |
| Fixed a bug whose root cause was hidden | `gotchas.md` + the module file |
| Found an existing entry to be wrong or stale | Update it, bump `Verified:`, log in `changelog.md` |

Rules:

- **Write in the repository** — knowledge that lives only in a conversation is lost.
- **Update, don't duplicate** — if an entry exists, refine it; never create a second copy.
- **Log every change** — append a dated line to `docs/knowledge/changelog.md`.
- **Keep `INDEX.md` accurate** — a new file that is not indexed does not exist for other agents.
- **Prune on change** — when a PR alters behaviour you documented, update the entry in the same change.

---

## Conflict Resolution

When two entries disagree:

1. Trust the **code**, not the documentation — verify at the cited location.
2. Trust the **newer `Verified:` date** when the code supports both.
3. Merge into one entry with the surviving fact; delete the loser.
4. Record the resolution in `changelog.md`.

---

## Freshness Model

| Status | Meaning | Action |
|---|---|---|
| `fresh` | Verified after the last change to the cited code | Use directly |
| `aging` | Not verified in a while, code may have moved | Verify before acting |
| `stale` | Known to be out of date | Do not trust; fix or delete |
| `superseded` | Replaced by a newer entry | Keep only as history pointer |

Mark status at the top of each knowledge file and in `INDEX.md`. An agent that finds a stale entry must fix it before finishing its task.

---

## Anti-Patterns

- **Knowledge hoarding** — keeping discoveries in the session instead of writing them down.
- **Documentation dumps** — pasting code or API signatures verbatim; link to `file:line` instead.
- **Copy-of-the-code** — duplicating what the source already says clearly; capture only the *why* and the *non-obvious*.
- **Unverified authority** — acting on an entry without checking the cited evidence.
- **Parallel truths** — creating `NOTES.md`, `LEARNINGS.md`, and `docs/knowledge/` all at once. One tree only.
- **Write-only knowledge** — creating entries nobody can find because `INDEX.md` was never updated.
- **Stale confidence** — leaving `high` confidence on an entry the code no longer supports.

---

## Checklist: Before Finishing Any Task

- [ ] Consulted `docs/knowledge/INDEX.md` before starting the investigation.
- [ ] Verified every knowledge entry I relied on against the actual code.
- [ ] Captured new durable findings in the correct knowledge file.
- [ ] Added or updated an entry in `INDEX.md` for anything new.
- [ ] Bumped `Verified:` and logged changes in `changelog.md`.
- [ ] Marked entries that my change invalidated as `stale` or updated them.
- [ ] Left no secrets, tokens, or private data in the knowledge base.
