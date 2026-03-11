# gsd-skills

A collection of GSD-compatible skills for [pi](https://github.com/badlogic/pi-coding-agent) / [gsd-pi](https://github.com/mariozechner/gsd-pi).

## Installation

Copy any skill directory into your global skills folder:

```bash
# Clone this repo
git clone https://github.com/djimenez18/gsd-skills.git

# Install all skills
cp -r gsd-skills/*/  ~/.gsd/agent/skills/

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

**Migrate any project's existing planning artifacts into GSD-2 format.**

Use when you have an existing project with planning history (BMAD epics, Notion docs, Jira tickets, markdown plans, sprint docs, or any other format) and you want to bring it into GSD-2 so `/gsd auto` can run cleanly.

**What it covers:**
- Auditing existing artifacts (BMAD, Notion, Jira, old GSD, sprint docs)
- Designing the milestone/slice/task structure
- Writing all GSD artifacts using the correct templates (roadmap, plan, task-plan, summary, project, requirements, decisions)
- Ensuring completed milestones have `SUMMARY.md` files so GSD routing works correctly
- Setting `depends_on` frontmatter so GSD starts at the right milestone
- Creating a housekeeping milestone for messy migrations (stale docs, oversized slices, placeholder tasks)
- Adding a full implementation audit slice (E1-E20 or equivalent) to verify the codebase matches the plan
- Running parser and routing validation checks before handing off to `/gsd auto`

**Distinct from `/gsd migrate`** which only ports the old GSD-1 `.planning/` format.

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
    references/     # optional reference docs loaded on-demand
```

Frontmatter requirements:
```yaml
---
name: your-skill          # must match directory name, lowercase + hyphens only
description: ...          # max 1024 chars, be specific about when to use it
---
```
