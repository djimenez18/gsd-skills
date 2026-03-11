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

**Onboard any project with existing planning history into GSD-2 so `/gsd auto` runs cleanly from the right milestone.**

Works with `.planning` directories, BMAD epics, Jira exports, Notion docs, sprint files, or any other planning format.

#### What it does

**Detects your situation** — checks what planning artifacts exist and whether [`jonathancostin/gsd-migrate`](https://github.com/jonathancostin/gsd-migrate) is installed, then takes the right path:

| Situation | Action |
|---|---|
| `.planning/` directory exists | Runs `/gsd migrate` (installs it if needed), then audits and fixes the output |
| BMAD `_bmad-output/` or epic files | Full manual migration using GSD templates |
| Jira / Notion / markdown plans | Full manual migration using GSD templates |
| Existing `.gsd/` that's messy or partial | Audit and fix in place |

**Applies post-migration hardening** regardless of which path was taken:

- Writes `SUMMARY.md` for all completed milestones — the #1 routing bug that causes GSD to activate the wrong milestone
- Sets `depends_on` frontmatter in `CONTEXT.md` so GSD starts at exactly the right milestone
- Detects and splits oversized slices (>5 active tasks) before enrichment so files land in final locations
- Rewrites placeholder task plans (`"Migrated from story: slug"`) into self-contained plans with Steps, Must-Haves, and Verification
- Creates a housekeeping milestone when docs are stale, DECISIONS.md is bullet-list format, or REQUIREMENTS.md is missing

**Runs a full implementation audit:**

- Cross-references every source story (BMAD epic, sprint item, etc.) against the actual codebase using `rg`
- Marks each as VERIFIED / PARTIAL / MISSING / DEFERRED
- Produces `AUDIT-REPORT.md` with a pre-production gate checklist
- Schedules any HIGH severity gaps as new tasks in the implementation backlog

**Validates before handoff:**

- Parser check — confirms all roadmap slice entries parse correctly
- Self-containment check — confirms every `T##-PLAN.md` has its own Steps and Verification (no stubs)
- Missing summary check — confirms no completed milestone is missing its `SUMMARY.md`
- Routing simulation — confirms GSD will activate the intended milestone first

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
