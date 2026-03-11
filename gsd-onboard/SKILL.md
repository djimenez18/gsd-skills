---
name: gsd-onboard
description: Migrate any project's existing planning artifacts into GSD-2. Handles .planning directories (calls /gsd migrate if installed), BMAD epics, Jira, Notion, or any other format. Applies post-migration fixes for routing correctness, self-contained task plans, oversized slice splitting, and full implementation audit. Use when starting GSD on a project with existing planning history.
---

You are onboarding an existing project into GSD-2 so that `/gsd auto` starts at exactly the right milestone, every task plan is self-contained, nothing from the original vision is lost, and the codebase is verified against the plan before launch.

Follow every phase in order. Do not skip validation steps.

---

## Phase 0: Detect What Exists

```bash
# What planning artifacts are present?
ls -la .planning/ 2>/dev/null && echo "HAS .planning"
ls -la .gsd/ 2>/dev/null && echo "HAS .gsd"
find . -maxdepth 3 -name "epics*.md" -o -name "sprint-status.yaml" \
       -o -name "*roadmap*.md" -o -name "prd.md" 2>/dev/null | grep -v node_modules

# Is /gsd migrate installed?
ls ~/.gsd/agent/extensions/gsd/migrate/ 2>/dev/null && echo "gsd-migrate INSTALLED" || echo "gsd-migrate NOT installed"
```

Based on findings, take the right path:

| Situation | Path |
|---|---|
| `.planning/` exists + `gsd-migrate` installed | **Path A** → run `/gsd migrate`, then Phase 3 |
| `.planning/` exists + NOT installed | Install first: `curl -fsSL https://raw.githubusercontent.com/jonathancostin/gsd-migrate/main/install.sh \| bash`, then Path A |
| BMAD `_bmad-output/` or epic files exist | **Path B** → manual migration, then Phase 3 |
| Other format (Jira export, Notion, markdown plans) | **Path B** → manual migration, then Phase 3 |
| `.gsd/` already exists (partial or messy) | **Path C** → audit and fix existing .gsd, then Phase 3 |

---

## Path A: Automated .planning Migration

**A.1 Run the migration**

```bash
/gsd migrate
```

Review the preview stats before confirming. The tool shows milestone count, slice count, task count, and completion percentages — verify these match your expectations before writing.

**A.2 Post-migration review checklist**

After the migration writes `.gsd/`, verify each of these:

1. **Structure** — `deriveState()` returns a coherent phase (not `pre-planning` unless truly empty). Active milestone/slice/task are sensible.
2. **Roadmap quality** — Slice entries have meaningful titles (not file paths or garbled text). `[x]`/`[ ]` markers match the old roadmap. Vision statement is present.
3. **Content spot-check** — Read 2-3 slice plans with the most tasks. Descriptions carry over meaningfully. Summary files exist for completed tasks.
4. **Requirements** — R-IDs are present and non-duplicate. Completed requirements are `validated`, in-progress are `active`.
5. **PROJECT.md** — Contains the real project description, not boilerplate.
6. **DECISIONS.md** — If written, contains extracted decisions (or is empty if none existed).

Fix any issues found in-place before continuing to Phase 3.

---

## Path B: Manual Migration from BMAD / Other Formats

**B.1 Read everything before designing anything**

```bash
# BMAD: read all planning artifacts
cat _bmad-output/planning-artifacts/epics.md
cat _bmad-output/planning-artifacts/prd.md
cat _bmad-output/implementation-artifacts/sprint-status.yaml
ls _bmad-output/planning-artifacts/
```

For each artifact: what is it, what's the completion state, is this the source of truth?

Record before proceeding:
- Project name and one-line description
- Tech stack with hard constraints (e.g. "no React Router", "motion/react not framer-motion")
- Done work (shipped epics/sprints/features)
- In-progress work (partial epics/sprints)
- Backlog work (not started)
- Key architectural decisions to preserve

**B.2 Design the milestone structure**

- One milestone = one major phase (MVP, Backend, Intelligence, Launch Readiness)
- Completed work → done milestones. Active work → current milestone.
- 4-20 slices per milestone
- **Max 5 active tasks per slice** — BMAD epics with 7+ stories MUST be split (see Phase 3.3)

Map source artifacts:

| Source | GSD equivalent |
|---|---|
| BMAD Epic (small, 3-5 stories) | 1-2 Slices |
| BMAD Epic (large, 6+ stories) | Multiple slices — split at natural concern boundary |
| BMAD Story | Task |
| Done sprint | Done slices with `[x]` in roadmap |
| ADR / decision | Row in DECISIONS.md |
| PRD requirement | Row in REQUIREMENTS.md |

**B.3 Write core GSD docs first (read templates before writing)**

```bash
cat ~/.gsd/agent/extensions/gsd/templates/project.md
cat ~/.gsd/agent/extensions/gsd/templates/requirements.md
cat ~/.gsd/agent/extensions/gsd/templates/decisions.md
```

- **PROJECT.md** — current state only. Tech stack with hard constraints explicit. Milestone sequence table. Reference REQUIREMENTS.md for capability contract.
- **REQUIREMENTS.md** — sections: `## Active`, `## Validated`, `## Deferred`, `## Out of Scope`, `## Traceability`. Each req: `### R001` header, class, owner (milestone/slice), status.
- **DECISIONS.md** — append-only table: `| # | When | Scope | Decision | Choice | Rationale | Revisable? |`. Extract from ADRs and planning docs.

**B.4 Write milestone roadmaps**

```bash
cat ~/.gsd/agent/extensions/gsd/templates/roadmap.md
```

Required sections: Vision, Success Criteria, Key Risks / Unknowns, Proof Strategy, Verification Classes, Milestone Definition of Done, Requirement Coverage, Slices, Boundary Map.

**Parser-critical slice format — must be exact:**
```
- [x] **S01: Title** `risk:medium` `depends:[]`
- [ ] **S02: Title** `risk:high` `depends:[S01]`
```
- `[x]` = done, `[ ]` = pending
- Title must be bold with colon: `**S01: Title**`
- Both tags required, backtick-wrapped: `` `risk:medium` `` `` `depends:[S01]` ``

**B.5 Write slice plans**

```bash
cat ~/.gsd/agent/extensions/gsd/templates/plan.md
```

Required sections: Goal, Demo, Must-Haves, Proof Level, Verification, Observability / Diagnostics, Integration Closure, Tasks, Files Likely Touched.

**Parser-critical task format — must be exact:**
```
- [ ] **T01: Title** `est:30m`
  - Why: ...
  - Files: `path/to/file`
  - Do: ...
  - Verify: ...
  - Done when: ...
```

**B.6 Write task plan files**

```bash
cat ~/.gsd/agent/extensions/gsd/templates/task-plan.md
```

Each task needs a standalone `tasks/T##-PLAN.md`. Use the full template:

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
- `path/file` — what this needs from prior work
## Expected Output
- `path/file` — what this produces
```

**The task plan must be fully self-contained.** Auto-mode inlines `T##-PLAN.md` as the primary execution contract in a fresh context. If it says "see slice plan for details", the agent has nothing to work with.

---

## Path C: Audit and Fix Existing .gsd

If `.gsd/` already exists (partial migration, old format, or messy state):

1. Read all milestone roadmaps — note which are complete (all `[x]`), which are partial
2. Read PROJECT.md, DECISIONS.md, REQUIREMENTS.md — note staleness
3. Spot-check 3 task plan files — are they self-contained or stubs?
4. Check task counts per backlog slice — any with >5 active tasks?
5. Note all issues and proceed through Phase 3 to fix them

---

## Phase 3: Post-Migration Hardening (ALL PATHS)

These steps apply after Path A, B, or C. They encode the hard-won lessons that most migrations miss.

### 3.1 Write SUMMARY.md for Every Completed Milestone

**This is the #1 routing bug.** Any milestone where all slices are `[x]` but no `SUMMARY.md` exists gets activated as "completing-milestone" — GSD asks the agent to write the summary instead of moving to the next milestone.

Find affected milestones:
```bash
node -e "
const fs = require('fs'), path = require('path');
fs.readdirSync('.gsd/milestones', {withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d => {
  const mid = d.name.match(/^(M\d+)/)?.[1]; if (!mid) return;
  const rp = path.join('.gsd/milestones', d.name, mid+'-ROADMAP.md');
  const sp = path.join('.gsd/milestones', d.name, mid+'-SUMMARY.md');
  if (!fs.existsSync(rp)) return;
  const roadmap = fs.readFileSync(rp,'utf-8');
  const slices = roadmap.match(/^- \[(x| )\] \*\*\w+:/gm)||[];
  const allDone = slices.length > 0 && slices.every(s=>s.includes('[x]'));
  if (allDone && !fs.existsSync(sp))
    console.log('NEEDS SUMMARY:', mid, '— all slices [x] but no SUMMARY.md');
});
"
```

For each milestone needing a summary:
```bash
cat ~/.gsd/agent/extensions/gsd/templates/milestone-summary.md
```

Write a real summary — not a stub. Key sections: What Happened (cross-slice narrative), Cross-Slice Verification, Forward Intelligence (what's fragile, what the next milestone should know, authoritative diagnostics).

### 3.2 Set depends_on Routing

GSD iterates M001 → M002 → M003 ... and activates the **first incomplete milestone with no unmet deps**. If you have old incomplete milestones that should wait for new work, set `depends_on` in their CONTEXT.md:

```markdown
---
depends_on:
  - M005
---

# M003 Context
...existing content...
```

Format rules: YAML frontmatter, 2-space indent for array items, milestone IDs uppercase.

The target state: old incomplete milestones blocked on new milestone → new milestone becomes active.

### 3.3 Detect and Split Oversized Slices

BMAD epics and old phases often map 1:1 to GSD slices but contain 7-10 tasks. Find them:

```bash
node -e "
const fs = require('fs'), path = require('path');
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(mid => {
  const slicesDir = path.join('.gsd/milestones', mid.name, 'slices');
  if (!fs.existsSync(slicesDir)) return;
  fs.readdirSync(slicesDir,{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(sid => {
    const plan = path.join(slicesDir, sid.name, sid.name+'-PLAN.md');
    if (!fs.existsSync(plan)) return;
    const active = (fs.readFileSync(plan,'utf-8').match(/^- \[ \] \*\*T\d+/gm)||[]).length;
    if (active > 5) console.log('OVERSIZED:', mid.name+'/'+sid.name, '—', active, 'active tasks');
  });
});
"
```

For each oversized slice:
1. Find the natural split line — distinct demo boundary or concern change
2. Create new slice directory (e.g. S30) with tasks/ subdirectory
3. Move task files past the split line into the new slice
4. Edit the original slice plan to remove moved tasks, update Goal/Demo scope
5. Write a new slice plan for the new slice
6. Add the new slice to the milestone roadmap

**Split before enriching task plans** — so enrichment writes to final file locations.

### 3.4 Enrich Placeholder Task Plans

Migrations often produce task stubs like `"Migrated from BMAD story: slug-name"`. Find them:

```bash
grep -r "Migrated from\|See slice plan\|TODO: fill" .gsd/milestones/*/slices/*/tasks/*.md -l
```

For each placeholder task plan, read the source story from the original planning artifact and rewrite with full Steps, Must-Haves, Verification, Observability Impact, Inputs, Expected Output.

### 3.5 Add a Housekeeping Milestone (if docs are messy)

If PROJECT.md is stale, DECISIONS.md is bullet-list format, REQUIREMENTS.md is missing, or task plans are placeholders — create a dedicated housekeeping milestone (e.g. M005: GSD Baseline) before the implementation milestone.

Housekeeping milestone slices:
- Fix core GSD docs (PROJECT.md, DECISIONS.md, REQUIREMENTS.md, STATE.md)
- Fix any roadmap/plan inconsistencies (task done but slice marked pending, etc.)
- Split oversized slices
- Enrich placeholder task plans
- Upgrade milestone roadmaps to full template (Key Risks, Proof Strategy, Verification Classes, DoD)
- Upgrade slice plans to full template (Proof Level, Verification, Observability, Integration Closure)
- Full implementation audit (see Phase 4)

Set all other incomplete milestones to `depends_on: [M_housekeeping]`.

---

## Phase 4: Implementation Audit

Before declaring the migration done, verify the codebase actually matches the plan.

Create an audit slice in the housekeeping milestone (or as a final slice in the current milestone) with these tasks:

**T01: Audit core features (epics 1 through N/2)**
- Read source epic/story files for each done slice
- `rg` the codebase for the key component or function per story
- Record: VERIFIED (rg confirms), PARTIAL (some ACs found), MISSING (nothing found), DEFERRED (not yet started)

**T02: Audit remaining features (epics N/2+1 through N)**
- Same approach for the second half of done slices
- For backlog epics: mark DEFERRED and note which M004/SXX slice they're scheduled in

**T03: Verify backlog task plan AC completeness**
- For each backlog task plan, open the source story
- Confirm every AC from the story is captured in the task plan — not just the headline
- Add any missing ACs to the task plan

**T04: Write AUDIT-REPORT.md**
- Location: `.gsd/milestones/M_housekeeping/AUDIT-REPORT.md`
- Per-epic results table: `| Epic | Story | Status | Evidence | Severity |`
- HIGH gaps (launch blockers) → add task to relevant M004 backlog slice
- MEDIUM gaps → note with recommendation
- DEFERRED items → confirm they have a scheduled M004 slice
- Pre-production gate checklist at bottom for human sign-off

---

## Phase 5: Validate Auto-Mode Readiness

Run all checks. Fix anything that fails before handing off to `/gsd auto`.

**5.1 Roadmap slice parser check**
```bash
node -e "
const fs = require('fs'), path = require('path');
const re = /^- \[(x| )\] \*\*(\w+): (.+?)\*\* \x60risk:(\w+)\x60 \x60depends:\[([^\]]*)\]\x60/gm;
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d=>{
  const mid = d.name.match(/^(M\d+)/)?.[1]; if(!mid) return;
  const f = path.join('.gsd/milestones',d.name,mid+'-ROADMAP.md');
  if(!fs.existsSync(f)) return;
  const content = fs.readFileSync(f,'utf-8');
  const slices=[]; let m;
  while((m=re.exec(content))!==null) slices.push(m[2]);
  console.log(mid+': '+slices.length+' slices parsed — '+(slices.join(', ')||'NONE'));
});
"
```
Every milestone roadmap must parse at least one slice.

**5.2 Task plan self-containment check**
```bash
node -e "
const fs = require('fs'), path = require('path');
let stubs=0, ok=0;
function check(dir) {
  if (!fs.existsSync(dir)) return;
  fs.readdirSync(dir,{withFileTypes:true}).forEach(d => {
    if (d.isDirectory()) check(path.join(dir,d.name));
    if (d.name.match(/^T\d+-PLAN\.md$/)) {
      const c = fs.readFileSync(path.join(dir,d.name),'utf-8');
      const good = (c.includes('## Steps')||c.includes('## Do')) &&
                   (c.includes('## Must-Haves')||c.includes('## Verify'));
      if (good) ok++; else { stubs++; console.log('STUB:',path.join(dir,d.name)); }
    }
  });
}
check('.gsd/milestones');
console.log('Self-contained:',ok,'Stubs:',stubs);
"
```
Must show `Stubs: 0`.

**5.3 Missing summary check**
```bash
node -e "
const fs = require('fs'), path = require('path');
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d=>{
  const mid = d.name.match(/^(M\d+)/)?.[1]; if(!mid) return;
  const rp = path.join('.gsd/milestones',d.name,mid+'-ROADMAP.md');
  const sp = path.join('.gsd/milestones',d.name,mid+'-SUMMARY.md');
  if(!fs.existsSync(rp)) return;
  const slices = fs.readFileSync(rp,'utf-8').match(/^- \[(x| )\] \*\*\w+:/gm)||[];
  const allDone = slices.length>0 && slices.every(s=>s.includes('[x]'));
  if(allDone && !fs.existsSync(sp))
    console.log('MISSING SUMMARY:',mid,'— GSD will activate this milestone to write its summary!');
});
"
```
Must produce no output.

**5.4 Routing simulation**
```bash
node -e "
const fs = require('fs'), path = require('path');
function resolveFile(dir,id,suffix) {
  if(!fs.existsSync(dir)) return null;
  const t=(id+'-'+suffix+'.md').toUpperCase();
  return fs.readdirSync(dir).find(e=>e.toUpperCase()===t)||null;
}
function rmf(mid,suffix) {
  const d=path.join('.gsd/milestones',mid);
  if(!fs.existsSync(d)) return null;
  const f=resolveFile(d,mid,suffix);
  return f?path.join(d,f):null;
}
function deps(mid) {
  const cf=rmf(mid,'CONTEXT'); if(!cf) return [];
  const c=fs.readFileSync(cf,'utf-8').trimStart();
  if(!c.startsWith('---')) return [];
  const rest=c.slice(c.indexOf('\n')+1);
  const end=rest.indexOf('\n---'); if(end===-1) return [];
  const lines=rest.slice(0,end).split('\n');
  const out=[]; let in_=false;
  for(const l of lines){
    if(l.trim()==='depends_on:'){in_=true;continue;}
    if(in_&&l.match(/^  - /)) out.push(l.replace('  - ','').trim().toUpperCase());
    else if(in_&&!l.match(/^  /)) in_=false;
  }
  return out;
}
function allDone(mid) {
  const rf=rmf(mid,'ROADMAP'); if(!rf) return false;
  const s=fs.readFileSync(rf,'utf-8').match(/^- \[(x| )\] \*\*\w+:/gm)||[];
  return s.length>0&&s.every(x=>x.includes('[x]'));
}
const milestones=fs.readdirSync('.gsd/milestones',{withFileTypes:true})
  .filter(d=>d.isDirectory()).map(d=>d.name.match(/^(M\d+)/)?.[1]).filter(Boolean).sort();
const done=new Set(milestones.filter(m=>allDone(m)&&rmf(m,'SUMMARY')));
let active=null;
for(const mid of milestones){
  const rf=rmf(mid,'ROADMAP'); if(!rf){console.log(mid+': no roadmap');continue;}
  if(allDone(mid)){
    const sf=rmf(mid,'SUMMARY');
    if(!sf&&!active){console.log(mid+': ⚠️  no SUMMARY → completing-milestone → ACTIVE (fix this!)');active=mid;break;}
    console.log(mid+': ✅ complete'); continue;
  }
  const unmet=deps(mid).filter(d=>!done.has(d));
  if(unmet.length>0){console.log(mid+': blocked on ['+unmet+'→ skip');continue;}
  console.log(mid+': ACTIVE ✓'); active=mid; break;
}
console.log('\nGSD starts with →',active);
"
```
Confirm the output shows the intended milestone as ACTIVE.

---

## Common Pitfalls

**Missing SUMMARY.md** — Any milestone with all slices `[x]` but no SUMMARY.md gets activated before your intended target. Always write summaries for shipped milestones.

**Stub task plans** — Auto-mode uses the task plan as its primary instructions. "See slice plan for details" gives the agent nothing. Every `T##-PLAN.md` must stand alone.

**Oversized slices** — BMAD/Jira epics map to 7-10 tasks. Split before enriching so enrichment writes to final file locations. Split rule: find the natural concern or demo boundary.

**Wrong roadmap format** — Parser is strict. Must be exact:
`- [ ] **S01: Title** \`risk:medium\` \`depends:[]\``

**Missing depends_on** — Without it, GSD picks the first incomplete milestone alphabetically. Add `depends_on: [M_new]` to all old incomplete milestones you don't want running yet.

**DECISIONS.md bullet format** — Template requires `| # | When | Scope | Decision | Choice | Rationale | Revisable? |` table. Convert bullets to rows.

**No REQUIREMENTS.md** — Task plans referencing R-IDs need the file to exist. Create it before writing task plans that reference requirement IDs.

**Enriching before splitting** — If you enrich task plans in S19 and then split S19 → S19+S30, the enriched T06/T07 files need to be moved. Always split first.
