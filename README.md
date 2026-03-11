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

**Onboard any project with existing planning history into GSD-2 so `/gsd auto` runs cleanly from the right milestone — with ground-truth completion markers, full vision coverage, self-contained task plans, and a test-fix loop on every slice.**

Works with `.planning` directories, BMAD epics, Jira exports, Notion docs, sprint files, or any other planning format.

#### What it does

**Detects your situation** — checks what planning artifacts exist and whether [`jonathancostin/gsd-migrate`](https://github.com/jonathancostin/gsd-migrate) is installed, then takes the right path:

| Situation | Action |
|---|---|
| `.planning/` directory exists | Runs `/gsd migrate` (installs if needed), then verifies ground truth and hardens output |
| BMAD `_bmad-output/` or epic files | Full manual migration using GSD templates |
| Jira / Notion / markdown plans | Full manual migration using GSD templates |
| Existing `.gsd/` that's messy or partial | Audit and fix in place |

**Verifies ground truth before trusting any completion marker:**

- Counts summary files vs plan files per phase — a phase is DONE only when every plan has a matching summary
- Cross-checks migrated `[x]`/`[ ]` markers against filesystem evidence — fixes any that are wrong
- Detects orphaned code (code written without GSD tracking) — verifies ACs and writes correct summaries or task plans
- Validates roadmap format with the parser immediately after writing each roadmap file

**Applies post-migration hardening:**

- Writes `SUMMARY.md` for all completed milestones — the #1 routing bug that activates the wrong milestone
- Sets `depends_on` frontmatter so GSD starts at exactly the right milestone
- Detects and splits oversized slices (>5 active tasks) before enrichment
- Enforces task plan self-containment at write-time — every `T##-PLAN.md` must have Steps, Must-Haves, and Verification before it's accepted
- Adds a "Test, Fix, and Confirm" task to every implementation slice — no slice is done until the full test suite passes

**Runs an audit-first implementation approach:**

- Adds a dedicated audit slice as the FIRST slice of the active milestone — no implementation begins until `IMPLEMENTATION-STATUS.md` is produced
- Per-story audit checks: code exists AND ACs fully covered AND wired into main pipeline AND test exists
- Marks each story VERIFIED / PARTIAL / MISSING with file:line evidence — not guesses
- Updates all downstream task plans: VERIFIED stories get skip notes, PARTIAL gaps get confirmed, MISSING stories get implementation tasks
- Vision coverage cross-check: confirms every source story has a corresponding GSD task (catches planning gaps before they become shipping gaps)

**Validates before handoff:**

- Parser check — all roadmap slice entries parse correctly
- Self-containment check — every `T##-PLAN.md` has Steps, Must-Haves, Verification (zero stubs)
- Missing summary check — no completed milestone is missing its `SUMMARY.md`
- Routing simulation — GSD activates the intended milestone
- Test-fix loop coverage — every pending slice has a fix-and-confirm task

#### Key improvements over v1

- **Ground truth verification** — filesystem beats migration heuristics; markers are checked not assumed
- **Orphaned code detection** — handles code written without GSD tracking (common when sessions end mid-task)
- **Audit-first** — `IMPLEMENTATION-STATUS.md` produced before any implementation begins; downstream tasks reference it
- **Per-story wiring check** — verifies code is connected to the main pipeline, not just that a file exists
- **Write-time self-containment enforcement** — checklist applied when writing task plans, not discovered at Phase 5
- **Test-fix loop on every slice** — implementation without a fix loop produces half-working features
- **Vision coverage cross-check** — catches BMAD stories that never made it into `.planning` phases

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
