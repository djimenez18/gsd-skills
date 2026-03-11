---
name: gsd-onboard
description: Migrate any project's existing planning artifacts (BMAD epics, Notion, Jira, markdown plans, sprint docs, or any other format) into fully GSD-2 compliant milestones with correct routing and auto-mode-ready task plans. Use when starting GSD on a project that already has planning history. Distinct from /gsd migrate which ports GSD-1 .planning/ format only.
---

You are migrating an existing project's planning artifacts into GSD-2 format. The goal is a `.gsd/` directory where `/gsd auto` starts at exactly the right milestone, every task plan is self-contained, and nothing from the original vision is lost.

This skill encodes hard-won lessons from a real migration. Follow every phase in order. Do not skip validation steps.

---

## Phase 1: Audit Existing Artifacts

Before writing a single GSD file, read everything that exists.

**1.1 Inventory the project**

Scan for planning artifacts:
```bash
find . -name "*.md" -path "*plan*" -o -name "*.md" -path "*epic*" -o \
       -name "*.md" -path "*story*" -o -name "*.yaml" -path "*sprint*" | \
       grep -v node_modules | grep -v .git
```

Also check for: `.planning/`, `_bmad-output/`, `docs/`, `ROADMAP.md`, `TODO.md`, Jira exports, Notion exports.

**1.2 Read everything before designing anything**

For each artifact found:
- What is this? (requirements, epics, stories, sprint status, architecture decisions, ADRs)
- What's the completion state? (done / in-progress / backlog)
- What's the source of truth? (if multiple files exist, which is most recent?)

**1.3 Extract the key facts**

Record these before proceeding:
- Project name and one-line description
- Tech stack (languages, frameworks, databases, key constraints)
- Done work: list of features/epics fully shipped
- In-progress work: list of features/epics partially done
- Backlog work: list of features/epics not started
- Any architectural decisions that must be preserved
- Any hard constraints (e.g. "no React Router", "use X not Y")

---

## Phase 2: Design the Milestone Structure

**2.1 Milestone sizing rules**

- One milestone = one major phase of the product lifecycle
- A milestone has 4-20 slices
- Typical structure: M001 = MVP, M002 = Backend hardening, M003 = Intelligence layer, M004 = Launch readiness, Mxxx = GSD housekeeping (if needed)
- Completed work goes into done milestones. Active work goes into the current milestone.

**2.2 Slice sizing rules**

- One slice = one demoable vertical increment
- **Maximum 5 active (unchecked) tasks per slice** — if a BMAD epic has 7+ stories, split it
- Split rule: find the natural concern boundary (e.g. "transactional emails" vs "admin campaign builder")
- Slice IDs are sequential within a milestone: S01, S02, S03...
- Done slices get `[x]` in the roadmap. Backlog slices get `[ ]`.

**2.3 Task sizing rules**

- One task = one context-window unit of work
- Maximum ~8 steps, ~10 files per task (the task-plan.md template warns at 10+/12+)
- Tasks execute sequentially: T01 completes before T02 starts
- **Every task plan must be self-contained** — auto-mode inlines T##-PLAN.md as the primary execution contract; the agent gets a fresh context

**2.4 Map source artifacts to GSD**

| Source format | GSD equivalent |
|---|---|
| BMAD Epic | Milestone (if large) or 2-4 Slices (if fits) |
| BMAD Story | Task |
| Sprint | Slice or group of slices |
| ADR / Decision | Row in DECISIONS.md |
| PRD requirement | Row in REQUIREMENTS.md |
| Done sprint | Done slices with `[x]` in roadmap |

---

## Phase 3: Create the GSD Artifacts

Read the template before writing each artifact:
```
~/.gsd/agent/extensions/gsd/templates/roadmap.md
~/.gsd/agent/extensions/gsd/templates/plan.md
~/.gsd/agent/extensions/gsd/templates/task-plan.md
~/.gsd/agent/extensions/gsd/templates/project.md
~/.gsd/agent/extensions/gsd/templates/requirements.md
~/.gsd/agent/extensions/gsd/templates/decisions.md
~/.gsd/agent/extensions/gsd/templates/milestone-summary.md
```

**3.1 Write PROJECT.md**

Current state only. Include:
- What the project is (one paragraph)
- Current milestone and completion state
- Tech stack with hard constraints called out explicitly
- Milestone sequence table (M001 ✅, M002 ✅, M003 🔧 active, etc.)
- Reference to REQUIREMENTS.md for capability contract

**3.2 Write REQUIREMENTS.md**

Sections: `## Active`, `## Validated`, `## Deferred`, `## Out of Scope`, `## Traceability`

Each requirement: `### R001` header, description, class (core-capability / primary-user-loop / compliance / operability / etc.), owner (milestone/slice), status.

**3.3 Write DECISIONS.md**

Append-only table format. Header comment: `<!-- Append-only. Never edit or remove existing rows. -->`

Table: `| # | When | Scope | Decision | Choice | Rationale | Revisable? |`

Extract from: ADRs, planning docs, "we decided to..." notes, architectural constraints. Aim for at least one decision per major milestone.

**3.4 Write milestone roadmaps**

For each milestone, write `M00X-ROADMAP.md` with all template sections:
```
# M001: Title
**Vision:** ...
## Success Criteria
## Key Risks / Unknowns
## Proof Strategy
## Verification Classes
## Milestone Definition of Done
## Requirement Coverage
## Slices
- [x] **S01: Title** `risk:medium` `depends:[]`
  > After this: ...
## Boundary Map
```

**Parser-critical format for slice entries — must be exact:**
```
- [ ] **S01: Title** `risk:low|medium|high` `depends:[]`
```
- Square brackets: `[ ]` for pending, `[x]` for done
- Bold title with colon after ID: `**S01: Title**`
- Backtick-wrapped risk tag: `` `risk:medium` ``
- Backtick-wrapped depends tag: `` `depends:[S01,S02]` `` or `` `depends:[]` ``

**3.5 Write slice plans**

For each backlog slice, write `S##-PLAN.md` with all template sections:
```
# S01: Title
**Goal:** ...
**Demo:** ...
## Must-Haves
## Proof Level
## Verification
## Observability / Diagnostics
## Integration Closure
## Tasks
- [ ] **T01: Title** `est:30m`
  - Why: ...
  - Files: `path/to/file`
  - Do: ...
  - Verify: ...
  - Done when: ...
## Files Likely Touched
```

**Parser-critical format for task entries — must be exact:**
```
- [ ] **T01: Title** `est:30m`
```

For done slices: they need a plan file (even minimal) with all tasks marked `[x]`, plus a SUMMARY.md.

**3.6 Write task plan files**

Each task needs a standalone `T##-PLAN.md` in the `tasks/` subdirectory. Use the full template:
```
---
estimated_steps: N
estimated_files: N
---

# T01: Title

**Slice:** S01 — Slice Title
**Milestone:** M001

## Description
## Steps
1. ...
## Must-Haves
- [ ] ...
## Verification
- command or observable check
## Observability Impact
- Signals added/changed: ...
- How a future agent inspects this: ...
- Failure state exposed: ...
## Inputs
- `path/file` — what this task needs from prior work
## Expected Output
- `path/file` — what this task produces
```

**The task plan must be self-contained.** The auto-mode agent gets a fresh context and receives `T##-PLAN.md` as the primary execution contract. If it says "see slice plan for details", the agent has no details.

---

## Phase 4: Write SUMMARY.md for All Completed Milestones

This is the step most migrations miss — and it breaks GSD routing if skipped.

**The rule:** Any milestone where all slices are `[x]` in the roadmap **must** have a `M00X-SUMMARY.md` file. If it doesn't, GSD activates it as "completing-milestone" (write the summary) instead of marking it complete and moving on.

For each fully-done milestone, write a summary using `milestone-summary.md` template:
```yaml
---
id: M001
provides:
  - feature or capability shipped
key_decisions:
  - decision made
patterns_established:
  - pattern introduced
observability_surfaces:
  - none / or specific surface
requirement_outcomes:
  - id: R001
    from_status: active
    to_status: validated
    proof: evidence
duration: N weeks
verification_result: passed
completed_at: YYYY-MM-DD
---
```

Body sections: What Happened, Cross-Slice Verification, Requirement Changes, Forward Intelligence (what fragile, what the next milestone should know, authoritative diagnostics, what assumptions changed), Files Created/Modified.

---

## Phase 5: Set Up Milestone Routing

GSD iterates milestones in alphabetical order (M001, M002, M003...) and activates the **first** one that is:
- Incomplete (not all slices `[x]`), AND
- Has no unmet `depends_on` deps in its CONTEXT.md

**If you have existing incomplete milestones that should not run before the new one:**

Add a `depends_on` frontmatter to their `M00X-CONTEXT.md`:
```yaml
---
depends_on:
  - M005
---

# M003 Context
...existing content...
```

This tells GSD: skip M003 until M005 is complete.

**The target routing state:**
- All historical milestones: all slices `[x]` + SUMMARY.md exists → skipped as complete
- Milestones that should wait: have `depends_on: [MTargetNew]` → skipped until target done
- New work milestone: no `depends_on` (or deps are met) + has incomplete slices → **ACTIVE**

---

## Phase 6: Add a GSD Housekeeping Milestone (if needed)

If the existing planning artifacts are messy (wrong format, stale docs, oversized slices, missing context), create a **standalone housekeeping milestone** before the implementation milestone. Name it something like `M00X: GSD Baseline & Orientation`.

This milestone's slices:
- **Fix core GSD docs** — rewrite PROJECT.md, convert DECISIONS.md to correct format, create REQUIREMENTS.md
- **Fix any inconsistencies** — roadmap says done but tasks aren't, or vice versa
- **Audit and split oversized slices** — any slice with >5 active tasks should be split before enrichment
- **Enrich backlog task plans** — replace placeholder descriptions with real file paths and ACs
- **Upgrade roadmaps/plans to full template** — add Key Risks, Proof Strategy, Verification Classes, DoD
- **Full implementation audit** — cross-reference all source stories against the codebase, produce AUDIT-REPORT.md

Make all other milestones `depends_on: [MHousekeeping]` so housekeeping runs first.

---

## Phase 7: Validate Auto-Mode Readiness

Run all checks before declaring the migration done.

**7.1 Parser check — roadmap slice format**
```bash
node -e "
const fs = require('fs'), path = require('path');
const re = /^- \[(x| )\] \*\*(\w+): (.+?)\*\* \x60risk:(\w+)\x60 \x60depends:\[([^\]]*)\]\x60/gm;
['M001','M002','M003','M004','M005'].forEach(mid => {
  const f = '.gsd/milestones/' + mid + '/' + mid + '-ROADMAP.md';
  if (!fs.existsSync(f)) return;
  const content = fs.readFileSync(f, 'utf-8');
  const slices = []; let m;
  while ((m = re.exec(content)) !== null) slices.push(m[2]);
  console.log(mid + ': ' + slices.length + ' slices parsed — ' + slices.join(', '));
});
"
```
Expected: all slices parse. If a slice doesn't appear, the format is wrong.

**7.2 Task plan self-containment check**
```bash
node -e "
const fs = require('fs'), path = require('path');
let stubs = 0, ok = 0;
function check(dir) {
  if (!fs.existsSync(dir)) return;
  fs.readdirSync(dir, {withFileTypes:true}).forEach(d => {
    if (d.isDirectory()) check(path.join(dir, d.name));
    if (d.name.match(/^T\d+-PLAN\.md$/)) {
      const c = fs.readFileSync(path.join(dir, d.name), 'utf-8');
      const selfContained = (c.includes('## Steps') || c.includes('## Do')) &&
                            (c.includes('## Must-Haves') || c.includes('## Verify'));
      if (selfContained) ok++; else { stubs++; console.log('STUB:', path.join(dir, d.name)); }
    }
  });
}
check('.gsd/milestones');
console.log('Self-contained: ' + ok + ', Stubs: ' + stubs);
"
```
Expected: `Stubs: 0`

**7.3 Oversized slice check**
```bash
node -e "
const fs = require('fs'), path = require('path');
fs.readdirSync('.gsd/milestones', {withFileTypes:true})
  .filter(d => d.isDirectory())
  .forEach(mid => {
    const slicesDir = path.join('.gsd/milestones', mid.name, 'slices');
    if (!fs.existsSync(slicesDir)) return;
    fs.readdirSync(slicesDir, {withFileTypes:true}).filter(d=>d.isDirectory()).forEach(sid => {
      const plan = path.join(slicesDir, sid.name, sid.name + '-PLAN.md');
      if (!fs.existsSync(plan)) return;
      const content = fs.readFileSync(plan, 'utf-8');
      const active = (content.match(/^- \[ \] \*\*T\d+/gm)||[]).length;
      if (active > 5) console.log('OVERSIZED: ' + mid.name + '/' + sid.name + ' — ' + active + ' active tasks');
    });
  });
console.log('Oversized check done');
"
```
Expected: no oversized slices.

**7.4 Summary file check — completed milestones**
```bash
node -e "
const fs = require('fs'), path = require('path');
const milestonesDir = '.gsd/milestones';
fs.readdirSync(milestonesDir, {withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d => {
  const mid = d.name.match(/^(M\d+)/)?.[1];
  if (!mid) return;
  const roadmapPath = path.join(milestonesDir, d.name, mid + '-ROADMAP.md');
  const summaryPath = path.join(milestonesDir, d.name, mid + '-SUMMARY.md');
  if (!fs.existsSync(roadmapPath)) return;
  const roadmap = fs.readFileSync(roadmapPath, 'utf-8');
  const slices = (roadmap.match(/^- \[(x| )\] \*\*\w+:/gm)||[]);
  const allDone = slices.length > 0 && slices.every(s => s.includes('[x]'));
  if (allDone && !fs.existsSync(summaryPath))
    console.log('MISSING SUMMARY: ' + mid + ' — all slices [x] but no SUMMARY.md → GSD will activate it!');
});
console.log('Summary check done');
"
```
Expected: no missing summaries.

**7.5 Milestone routing simulation**
```bash
node -e "
const fs = require('fs'), path = require('path');

function resolveFile(dir, idPrefix, suffix) {
  if (!fs.existsSync(dir)) return null;
  const target = (idPrefix + '-' + suffix + '.md').toUpperCase();
  return fs.readdirSync(dir).find(e => e.toUpperCase() === target) || null;
}
function resolveMilestoneFile(mid, suffix) {
  const mDir = path.join('.gsd/milestones', mid);
  if (!fs.existsSync(mDir)) return null;
  return resolveFile(mDir, mid, suffix) ? path.join(mDir, resolveFile(mDir, mid, suffix)) : null;
}
function parseContextDeps(content) {
  if (!content) return [];
  const trimmed = content.trimStart();
  if (!trimmed.startsWith('---')) return [];
  const rest = trimmed.slice(trimmed.indexOf('\n') + 1);
  const end = rest.indexOf('\n---');
  if (end === -1) return [];
  const lines = rest.slice(0, end).split('\n');
  const deps = []; let inDeps = false;
  for (const l of lines) {
    if (l.trim() === 'depends_on:') { inDeps = true; continue; }
    if (inDeps && l.match(/^  - /)) deps.push(l.replace('  - ','').trim().toUpperCase());
    else if (inDeps && !l.match(/^  /)) inDeps = false;
  }
  return deps;
}
function allSlicesDone(roadmap) {
  const slices = (roadmap.match(/^- \[(x| )\] \*\*\w+:/gm)||[]);
  return slices.length > 0 && slices.every(s => s.includes('[x]'));
}

const milestones = fs.readdirSync('.gsd/milestones', {withFileTypes:true})
  .filter(d => d.isDirectory())
  .map(d => d.name.match(/^(M\d+)/)?.[1]).filter(Boolean).sort();

const completeSet = new Set();
for (const mid of milestones) {
  const rf = resolveMilestoneFile(mid, 'ROADMAP');
  const sf = resolveMilestoneFile(mid, 'SUMMARY');
  if (rf && allSlicesDone(fs.readFileSync(rf,'utf-8')) && sf) completeSet.add(mid);
}

let activeMilestone = null;
for (const mid of milestones) {
  const rf = resolveMilestoneFile(mid, 'ROADMAP');
  if (!rf) { console.log(mid + ': no roadmap → skip'); continue; }
  const roadmap = fs.readFileSync(rf, 'utf-8');
  const done = allSlicesDone(roadmap);
  const sf = resolveMilestoneFile(mid, 'SUMMARY');
  if (done && !sf && !activeMilestone) {
    console.log(mid + ': ⚠️  all slices done but NO SUMMARY → completing-milestone → ACTIVE (wrong!)');
    activeMilestone = mid; break;
  }
  if (done && sf) { console.log(mid + ': ✅ complete → skip'); continue; }
  const cf = resolveMilestoneFile(mid, 'CONTEXT');
  const deps = parseContextDeps(cf ? fs.readFileSync(cf,'utf-8') : null);
  const unmet = deps.filter(d => !completeSet.has(d));
  if (unmet.length > 0) { console.log(mid + ': deps unmet [' + unmet + '] → skip'); continue; }
  console.log(mid + ': ✓ ACTIVE');
  activeMilestone = mid; break;
}
console.log('\nGSD will start with:', activeMilestone);
"
```
Expected: GSD will start with the milestone you intend.

---

## Common Pitfalls

**Pitfall 1: Task stubs**
Writing task plan files that say "see slice plan for details." Auto-mode inlines the task plan as the primary execution contract. If it has no detail, the agent has no instructions. Every task plan must have Steps, Must-Haves, and Verification in the file itself.

**Pitfall 2: Missing SUMMARY.md for done milestones**
Any milestone with all slices `[x]` but no SUMMARY.md gets activated as "completing-milestone" — GSD asks the agent to write the summary instead of moving to the next milestone. Write stub summaries for all shipped milestones.

**Pitfall 3: Oversized slices from 1:1 epic mapping**
BMAD/Jira epics map to 7-10 stories. GSD slices should be 3-5 tasks. Always check task counts and split before enriching, so enrichment writes to final file locations.

**Pitfall 4: Missing depends_on for old milestones**
If M003 is incomplete and you want GSD to run M005 first, M003 must have `depends_on: [M005]` in its CONTEXT.md frontmatter. Without it, GSD picks M003 as active regardless.

**Pitfall 5: Wrong roadmap slice format**
The GSD parser is strict. These all fail:
- `- [ ] S01: Title \`risk:medium\`` — missing `**bold**` around title
- `- [ ] **S01 Title** \`risk:medium\` \`depends:[]\`` — missing colon after ID
- `- [ ] **S01: Title** risk:medium depends:[]` — missing backtick wrappers
This must be exact: `- [ ] **S01: Title** \`risk:medium\` \`depends:[]\``

**Pitfall 6: No REQUIREMENTS.md**
M004+ task plans that reference R-IDs (R015, R022, etc.) in their Requirement Coverage sections need a REQUIREMENTS.md to exist. If it doesn't, create it early in the housekeeping milestone so later tasks can reference real IDs.

**Pitfall 7: DECISIONS.md bullet format**
The GSD template requires a `| # | When | Scope | Decision | Choice | Rationale | Revisable? |` table. Many migrations bring in bullet lists from old planning docs. Convert to table format — the append-only constraint means you never delete, only add rows.

---

## Checklist Before Handing Off to `/gsd auto`

- [ ] All templates read before writing artifacts
- [ ] PROJECT.md reflects current state (not aspirational or historical)
- [ ] REQUIREMENTS.md exists with R-IDs for all active capabilities
- [ ] DECISIONS.md is a table (not bullets), all historical decisions migrated
- [ ] All completed milestones have SUMMARY.md files
- [ ] All milestone roadmaps have complete template sections (Vision, Success Criteria, Key Risks, Proof Strategy, Verification Classes, DoD, Requirement Coverage, Slices, Boundary Map)
- [ ] No slice has >5 active tasks
- [ ] All task plan files are self-contained (Steps/Must-Haves/Verification in the file)
- [ ] CONTEXT.md `depends_on` set correctly so GSD starts at the right milestone
- [ ] Parser check passes (all slice IDs parse from roadmaps)
- [ ] Routing simulation shows correct starting milestone
- [ ] Git commit with backup tag before running auto
