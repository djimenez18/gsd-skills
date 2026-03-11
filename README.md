# gsd-skills

A collection of GSD-compatible skills for [pi](https://github.com/badlogic/pi-coding-agent) / [gsd-pi](https://github.com/mariozechner/gsd-pi).

## Installation

```bash
# Clone this repo
git clone https://github.com/djimenez18/gsd-skills.git

# Install all skills
cp -r gsd-skills/*/ ~/.gsd/agent/skills/

# Or install a specific skill
cp -r gsd-skills/gsd-onboard ~/.gsd/agent/skills/
```

Then use it in pi:
```
/skill:gsd-onboard
```

Or pi will auto-load it when your task matches the description.

---

## Skills

### `gsd-onboard`

**Onboard any project with existing planning history into GSD-2 so `/gsd auto` runs cleanly from exactly the right milestone.**

Works with any project and any planning format — `.planning` directories, BMAD epics, GitHub issues, Jira exports, Notion docs, sprint files, a plain README, or no planning docs at all.

#### What it does

**Detects your situation** — checks what planning artifacts and code exist, identifies the test runner and type check commands, discovers installed skills and tool integrations, then takes the right path:

| Situation | Path |
|---|---|
| `.planning/` directory exists + `gsd-migrate` installed | **Path A** — automated migration |
| `.planning/` exists, `gsd-migrate` not installed | Installs it, then Path A |
| Any planning docs (epics, PRD, backlog, issues) | **Path B** — manual migration |
| `.gsd/` already exists but is messy or partial | **Path C** — audit and fix |
| No planning docs at all | **Path D** — bootstrap from scratch |

**Verifies ground truth before trusting any completion marker:**

- Counts summary files vs plan files per phase — DONE only when every plan has a matching summary
- Cross-checks migrated `[x]`/`[ ]` markers against filesystem evidence and fixes wrong ones
- Detects orphaned code (code written without GSD tracking) — checks ACs and writes correct summaries or task plans
- Validates roadmap format with the parser immediately after each roadmap file is written

**Applies post-migration hardening:**

- Writes `MILESTONE-SUMMARY.md` for completed milestones — the #1 routing bug that activates the wrong milestone
- Sets `depends_on` frontmatter so GSD starts at exactly the right milestone
- Detects and splits oversized slices (> 5 active tasks) before enrichment so files land in final locations
- Enforces task plan self-containment at write-time — every `T##-PLAN.md` must have Steps, Must-Haves, and Verification
- Adds a "Test, Fix, and Confirm" task to every implementation slice using the project's actual test runner

**Preserves skills, agents, and tool integrations:**

- Discovers installed skills (GSD, BMAD, custom) and configures `.gsd/preferences.md` so GSD auto can find and use them
- Documents agent manifests and MCP server integrations in preferences so they're available during execution
- Configures multi-agent skills (like party mode) for **auto-approve during GSD auto** — generates multi-perspective deliberation, synthesizes consensus, continues executing without waiting for user input
- Uses absolute `~/` paths for skills outside `~/.gsd/agent/skills/` — bare names only resolve from two directories

**Runs an audit-first implementation approach:**

- Adds a dedicated audit slice as the first pending slice — no implementation until `IMPLEMENTATION-STATUS.md` is produced
- Per-requirement audit: code exists AND ACs fully covered AND wired into main execution path AND test exists
- Marks each requirement VERIFIED / PARTIAL / MISSING with file:line evidence
- Updates downstream task plans: VERIFIED requirements get skip notes, PARTIAL gaps get confirmed, MISSING requirements get implementation tasks
- Vision coverage cross-check: confirms every source requirement has a GSD task (catches planning gaps before they ship as missing features)

**Validates before handoff:**

- Roadmap parser check — all slice entries parse correctly
- Self-containment check — every `T##-PLAN.md` has Steps, Must-Haves, Verification
- Missing summary check — no complete milestone is missing its summary
- Routing simulation — confirms GSD activates the intended milestone
- Test-fix loop coverage — every pending slice has a fix-and-confirm task

#### Works with any stack

The skill discovers your project's test runner and type check commands in Phase 0 and uses them consistently — not hardcoded Node.js/TypeScript commands. Works the same for Python + pytest, Go + go test, Rust + cargo test, or any other stack.

#### Party mode + GSD auto

If your project uses multi-agent skills (like BMAD party mode), the skill configures them for **non-interactive deliberation** during GSD auto. The agent loads the agent manifest, generates perspectives from 2-3 relevant agents, synthesizes the consensus, documents the decision in the task summary, and keeps executing — no pause for user input. During interactive sessions, party mode runs normally with full conversation flow.

#### Related

- [`jonathancostin/gsd-migrate`](https://github.com/jonathancostin/gsd-migrate) — GSD-2 extension that adds `/gsd migrate` for automated `.planning` → `.gsd` conversion. `gsd-onboard` calls this when available and applies hardening on top.

---

## Adding Skills to This Repo

Each skill lives in its own directory with a `SKILL.md` following the [Agent Skills standard](https://agentskills.io/specification):

```
gsd-skills/
  gsd-onboard/
    SKILL.md
  your-skill/
    SKILL.md
    scripts/        # optional helper scripts
    references/     # optional reference docs
```

Frontmatter requirements:
```yaml
---
name: your-skill          # must match directory name, lowercase + hyphens only
description: ...          # max 1024 chars — be specific about when to use it
---
```
