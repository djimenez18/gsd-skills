---
name: gsd-onboard
description: Migrate any project's existing planning artifacts into GSD so that `/gsd auto` starts at exactly the right milestone, every task plan is self-contained, the full vision is covered, and the codebase is verified against the plan before launch.
---

You are onboarding an existing project into GSD. Your job is to produce a `.gsd/` directory where `/gsd auto` activates at precisely the right milestone with ground-truth completion markers, self-contained task plans, and zero blind spots against the original vision.

Follow every phase in order. Do not skip any step.

---

## Phase 0: Detect What Exists

```bash
ls -la .planning/ 2>/dev/null && echo "HAS .planning"
ls -la .gsd/ 2>/dev/null && echo "HAS .gsd"
find . -maxdepth 3 \( -name "epics*.md" -o -name "sprint-status.yaml" \
     -o -name "*roadmap*.md" -o -name "prd.md" \) 2>/dev/null | grep -v node_modules
ls ~/.gsd/agent/extensions/gsd/migrate/ 2>/dev/null && echo "gsd-migrate INSTALLED" || echo "NOT installed"
```

| Situation | Path |
|---|---|
| `.planning/` + `gsd-migrate` installed | **Path A** — automated migration |
| `.planning/` + NOT installed | Install: `curl -fsSL https://raw.githubusercontent.com/jonathancostin/gsd-migrate/main/install.sh \| bash`, then Path A |
| BMAD `_bmad-output/` or epic files | **Path B** — manual migration |
| Other format (Jira, Notion, markdown) | **Path B** — manual migration |
| `.gsd/` already exists (partial/messy) | **Path C** — audit and fix |

---

## Path A: Automated .planning Migration

### A.1 Run the migration

```bash
/gsd migrate
```

Review preview stats before confirming. Verify milestone count, slice count, task count, and completion percentages look plausible before writing.

### A.2 Immediately run the roadmap parser on the first written roadmap

Do this before anything else:

```bash
node -e "
const fs = require('fs');
const re = /^- \[(x| )\] \*\*(\w+): (.+?)\*\* \x60risk:(\w+)\x60 \x60depends:\[([^\]]*)\]\x60/gm;
const content = fs.readFileSync('.gsd/milestones/M001/M001-ROADMAP.md','utf-8');
let count=0, m;
while((m=re.exec(content))!==null){count++;console.log('OK:',m[2],m[1]==='x'?'[done]':'[pending]');}
if(count===0) throw new Error('ZERO slices parsed — roadmap format is wrong, fix before continuing');
console.log('Parsed',count,'slices — format OK');
"
```

If count is 0: stop. Fix the roadmap format before writing any more files.

### A.3 Ground Truth Verification — filesystem beats migration heuristics

**This is the most important step.** Do not trust the migration output's `[x]`/`[ ]` markers. Verify completion from the actual filesystem.

```bash
# Count summary files vs plan files per phase
for phase in .planning/phases/*/; do
  plans=$(ls "$phase"*-PLAN.md 2>/dev/null | wc -l)
  summaries=$(ls "$phase"*-SUMMARY.md 2>/dev/null | wc -l)
  name=$(basename "$phase")
  if [ "$summaries" -eq "$plans" ] && [ "$plans" -gt 0 ]; then
    echo "DONE     $name ($summaries/$plans)"
  elif [ "$summaries" -gt 0 ]; then
    echo "PARTIAL  $name ($summaries/$plans)"
  else
    echo "PENDING  $name ($summaries/$plans)"
  fi
done
```

Rules:
- Phase is **DONE** only if `summaries == plans` (every plan has a matching summary)
- Phase is **PARTIAL** if `0 < summaries < plans`
- Phase is **PENDING** if `summaries == 0`

Cross-check against the migrated roadmap `[x]`/`[ ]` markers. If they differ: **fix the roadmap manually before continuing.** Wrong completion markers cause GSD auto to activate the wrong milestone.

### A.4 Detect Orphaned Code (code exists but no summary)

For every PENDING or PARTIAL phase, check whether implementing code actually exists in the codebase:

```bash
# For each pending phase, read its PLAN files to find expected output files
# then check if those files exist in src/
for phase in .planning/phases/*/; do
  summaries=$(ls "$phase"*-SUMMARY.md 2>/dev/null | wc -l)
  plans=$(ls "$phase"*-PLAN.md 2>/dev/null | wc -l)
  if [ "$summaries" -lt "$plans" ]; then
    echo "=== $(basename $phase) — checking for orphaned code ==="
    # Read plan files to find expected output files, then grep src/
    grep -h "output_files:\|creates:\|produces:" "$phase"*.md 2>/dev/null | head -5
  fi
done
```

When you find orphaned code (code exists, summary missing):
- Read the BMAD story ACs for that phase
- Search for implementing code: `rg "keyFunction|ComponentName" src/ -l`
- **If code fully covers ACs AND tests exist**: write a SUMMARY.md marking it done
- **If code partially covers ACs**: mark PARTIAL — write task plan for remaining ACs only
- **If code is a skeleton/stub**: mark PENDING — write a full task plan

### A.5 Post-migration review checklist

1. `deriveState()` returns a coherent active milestone (not `pre-planning` unless truly empty)
2. Slice entries have meaningful titles — not file paths or garbled text
3. `[x]`/`[ ]` markers match filesystem ground truth from A.3
4. Read 2-3 task plans — they must be self-contained (see Phase 3.4)
5. PROJECT.md contains the real project description
6. DECISIONS.md contains extracted decisions (or is empty)

---

## Path B: Manual Migration from BMAD / Other Formats

### B.1 Read everything before designing anything

```bash
cat _bmad-output/planning-artifacts/epics.md
cat _bmad-output/planning-artifacts/prd.md
cat _bmad-output/implementation-artifacts/sprint-status.yaml 2>/dev/null
ls _bmad-output/planning-artifacts/
```

Record before proceeding:
- Project name and one-line description
- Tech stack with hard constraints
- Done work (has summary files or verified shipped)
- In-progress work (partial)
- Backlog work (not started)
- Key architectural decisions to preserve

### B.2 Establish ground truth from filesystem

Before designing any milestone structure, verify completion from the filesystem using the same technique as A.3. Never design the GSD structure from planning docs alone — always verify against actual code and summary files.

### B.3 Design the milestone structure

- One milestone = one major phase (MVP, Backend, Intelligence, Launch)
- Completed work → done milestones with `[x]` slices and SUMMARY.md files
- Active work → current milestone
- **Max 5 active tasks per slice** — large epics MUST be split (see Phase 3.3)

Map source artifacts:

| Source | GSD equivalent |
|---|---|
| BMAD Epic (3-5 stories) | 1-2 Slices |
| BMAD Epic (6+ stories) | Multiple slices — split at concern boundary |
| BMAD Story | Task |
| Done sprint | Done slices `[x]` with SUMMARY.md |
| ADR / decision | Row in DECISIONS.md |
| PRD requirement | Row in REQUIREMENTS.md |

### B.4 Write core GSD docs first

```bash
cat ~/.gsd/agent/extensions/gsd/templates/project.md
cat ~/.gsd/agent/extensions/gsd/templates/requirements.md
cat ~/.gsd/agent/extensions/gsd/templates/decisions.md
```

- **PROJECT.md** — current state only. Tech stack with hard constraints explicit.
- **REQUIREMENTS.md** — sections: Active, Validated, Deferred, Out of Scope, Traceability
- **DECISIONS.md** — append-only table: `| # | When | Scope | Decision | Choice | Rationale | Revisable? |`

### B.5 Write milestone roadmaps

```bash
cat ~/.gsd/agent/extensions/gsd/templates/roadmap.md
```

**Parser-critical slice format — must be exact:**
```
- [x] **S01: Title** `risk:medium` `depends:[]`
- [ ] **S02: Title** `risk:high` `depends:[S01]`
```

**After writing each roadmap, immediately validate:**
```bash
node -e "
const fs = require('fs');
const re = /^- \[(x| )\] \*\*(\w+): (.+?)\*\* \x60risk:(\w+)\x60 \x60depends:\[([^\]]*)\]\x60/gm;
const f = process.argv[1];
const content = fs.readFileSync(f,'utf-8');
let count=0, m;
while((m=re.exec(content))!==null){count++;console.log('OK:',m[2]);}
if(count===0) throw new Error('ZERO slices parsed in '+f+' — fix format before continuing');
console.log(count,'slices parsed OK');
" .gsd/milestones/M001/M001-ROADMAP.md
```

Do this for every roadmap immediately after writing it. Do not write slice files until the roadmap parses correctly.

### B.6 Write slice plans

```bash
cat ~/.gsd/agent/extensions/gsd/templates/plan.md
```

Required sections: Goal, Demo, Must-Haves, Proof Level, Verification, Observability / Diagnostics, Integration Closure, Tasks, Files Likely Touched.

**Parser-critical task format:**
```
- [ ] **T01: Title** `est:30m`
  - Why: ...
  - Files: `path/to/file`
  - Do: ...
  - Verify: ...
  - Done when: ...
```

**Every slice must end with a "Test, Fix, and Confirm" task.** This task:
- Runs the full vitest suite for all modules touched in the slice
- Runs `pnpm typecheck` from repo root
- Fixes every failure before marking the task done
- For UI slices: browser-verifies every surface with no console errors
- Runs full monorepo vitest to confirm no regressions
- Updates IMPLEMENTATION-STATUS.md marking slice stories as VERIFIED

### B.7 Write task plan files

```bash
cat ~/.gsd/agent/extensions/gsd/templates/task-plan.md
```

**Before writing each task plan, confirm it will have ALL of these:**
- [ ] `## Description` — what this builds, referencing the BMAD story number
- [ ] `## Steps` — numbered, specific, executable (not "implement X" — say HOW)
- [ ] `## Must-Haves` — checkbox list drawn directly from the BMAD story ACs
- [ ] `## Verification` — exact commands or browser actions (not "verify it works")
- [ ] `## Inputs` — specific file paths this task reads
- [ ] `## Expected Output` — specific file paths this task produces or modifies

**The self-containment test:** Cover the slice plan. Can GSD auto execute this task with only the task plan file and the repo? If no → rewrite.

**If a task plan says "see slice plan for details" anywhere → reject it and rewrite.**

For tasks implementing BMAD stories:
- Always reference the story by number: "BMAD Story 4.2 (epics.md ~line 831)"
- Tell GSD auto to READ the story ACs first as step 1
- Include every AC as a Must-Have checkbox

---

## Path C: Audit and Fix Existing .gsd

1. Run the ground truth verification (same as A.3) — compare filesystem to roadmap markers
2. Read all milestone roadmaps — validate with parser check (same as A.2) for each one
3. Check every roadmap for the "all `[x]` but no SUMMARY.md" bug (see Phase 3.1)
4. Spot-check 3 task plans — are they self-contained per B.7 checklist?
5. Check task counts per pending slice — any with > 5 active tasks? (see Phase 3.3)
6. Note all issues and proceed through Phase 3 to fix them

---

## Phase 3: Post-Migration Hardening (ALL PATHS)

### 3.1 Write SUMMARY.md for Every Completed Milestone

Any milestone where all slices are `[x]` but no `SUMMARY.md` exists gets activated as "completing-milestone" — GSD auto writes the summary instead of moving on.

```bash
node -e "
const fs = require('fs'), path = require('path');
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d => {
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

Write a real summary — not a stub. Key sections: What Happened (cross-slice narrative), Cross-Slice Verification, Forward Intelligence (what's fragile, what the next milestone needs to know, authoritative diagnostics).

### 3.2 Set depends_on Routing

GSD activates the **first incomplete milestone with no unmet deps**. Block old incomplete milestones from running before new ones:

```yaml
---
depends_on:
  - M005
---
# M003 Context
```

Format: YAML frontmatter, 2-space indent for array items, milestone IDs uppercase.

### 3.3 Detect and Split Oversized Slices

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

**Split before enriching task plans** — so enrichment writes to final file locations.

To split: find the natural concern boundary, create a new slice directory, move tasks past the boundary, update both slice plans, add the new slice to the milestone roadmap.

### 3.4 Enforce Task Plan Self-Containment

Find stubs:
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
      const good = c.includes('## Steps') && c.includes('## Must-Haves') && c.includes('## Verification');
      if (good) ok++; else { stubs++; console.log('STUB:', path.join(dir,d.name)); }
    }
  });
}
check('.gsd/milestones');
console.log('Self-contained:', ok, '  Stubs:', stubs);
"
```

Must show `Stubs: 0` before Phase 4.

### 3.5 Add Test-Fix Loop to Every Slice

Every implementation slice must have a final "Test, Fix, and Confirm" task. Check for it:

```bash
for slice in .gsd/milestones/*/slices/*/; do
  plan="${slice}$(basename $slice)-PLAN.md"
  [ -f "$plan" ] || continue
  # Check if all tasks in the plan are [x] (slice is done — skip)
  pending=$(grep -c "^- \[ \]" "$plan" 2>/dev/null || echo 0)
  [ "$pending" -eq 0 ] && continue
  if ! grep -qi "test.*fix.*confirm\|fix.*verify\|fix and confirm\|fix, and confirm" "$plan"; then
    echo "MISSING fix-and-test task: $plan"
  fi
done
```

For each slice missing it: add a final T## task following this template:

```markdown
- [ ] **T##: Test, Fix, and Confirm** `est:2h`
  - Why: No slice is done until every test passes and every surface is verified
  - Files: All files modified in this slice
  - Do: Run vitest for slice modules → fix all failures → re-run until GREEN. Run pnpm typecheck → fix all errors → re-run until exits 0. For UI slices: browser-verify every surface, zero console errors. Run full monorepo vitest — fix any regressions. Update IMPLEMENTATION-STATUS.md
  - Verify: pnpm vitest run exits 0; pnpm typecheck exits 0; no browser console errors
  - Done when: Zero failures anywhere; all slice stories marked VERIFIED in IMPLEMENTATION-STATUS.md
```

### 3.6 Detect Orphaned Code

For all pending phases or slices, check if implementing code already exists in the codebase before writing task plans from scratch:

```bash
# Find key function/type names from plan files, search src/
grep -h "creates\|produces\|output.*file\|key_files" .planning/phases/PHASENAME/*.md 2>/dev/null | \
  grep -oE '[a-zA-Z]+\.(ts|tsx|md)' | sort -u | while read f; do
    found=$(find . -name "$f" -not -path "*/node_modules/*" 2>/dev/null | head -1)
    [ -n "$found" ] && echo "EXISTS: $found"
  done
```

When orphaned code is found:
- Read the BMAD story ACs for that phase
- Check each AC: is it implemented? is there a test?
- VERIFIED (all ACs + tests): write SUMMARY.md, mark done
- PARTIAL (some ACs): write task plan covering only the missing ACs
- STUB/SKELETON: write full task plan from scratch

---

## Phase 4: Implementation Audit

Before any new implementation begins, produce `IMPLEMENTATION-STATUS.md` — a ground-truth table of every source story verified against the actual codebase. All downstream task plans reference this document.

### 4.1 Create the audit slice

Add a dedicated audit slice as the **first slice** of the active milestone (push existing slices down). This slice produces `IMPLEMENTATION-STATUS.md` and updates all downstream task plans before a single line of implementation is written.

Audit slice tasks:
- **T01: Audit Epics 1 through N/2** — working notes for first half of stories
- **T02: Audit Epics N/2+1 through N** — working notes for second half
- **T03: Audit wiring (not just file existence)** — for each story, check not just that the function exists but that it's connected to the main pipeline (imported in index.ts, gateway.ts, executor.ts, or cron-jobs.ts as appropriate)
- **T04: Write IMPLEMENTATION-STATUS.md and update downstream plans**

### 4.2 Per-story audit template

For each story, follow this exact process:

1. Read the story's **Given/When/Then** acceptance criteria from the source epic
2. Identify the **primary artifact** — the function, file, route, or component that implements the core AC
3. Search: `rg "primaryArtifact" src/ --include="*.ts" --include="*.tsx" -l`
4. If found: read the file, confirm it covers **each AC fully** — not just that it exists
5. Check **wiring**: is it imported/called from the main execution path?
6. Check **test**: `rg "primaryArtifact" src/ --include="*.test.ts" -l` — does a test exist that would fail if this AC were removed?
7. Record status:
   - **VERIFIED**: code found + all ACs covered + wired into pipeline + test exists
   - **PARTIAL**: code exists but one or more ACs missing, OR not wired, OR no test
   - **MISSING**: no implementing code found

Severity for gaps:
- **HIGH**: user-facing feature that blocks a demo or the stated "operator can..." outcome
- **MEDIUM**: feature works but with degraded behavior or missing edge case
- **LOW**: observability, logging, or cosmetic gap

### 4.3 IMPLEMENTATION-STATUS.md format

```markdown
# Project Name — Implementation Status

**Audited:** {date}
**Total stories:** N
**VERIFIED:** N | **PARTIAL:** N | **MISSING:** N

## Epic 1: Epic Title
| Story | Title | Status | Evidence | Gap |
|-------|-------|--------|----------|-----|
| 1.1 | Story Title | VERIFIED | src/bridge/translate.ts:L12 + translate.test.ts | none |
| 1.2 | Story Title | PARTIAL  | src/intent/router.ts exists but embedding fallback (AC3) untested | Missing test for AC3 |
| 1.3 | Story Title | MISSING  | rg finds no /djclaw bot.command registration | Full implementation needed |

## Epic 2: ...
[continue for all epics]

## Summary
### MISSING (build from scratch)
- Story X.X: Title — gap → covered by M00N/S##/T##
### PARTIAL (complete these)
- Story X.X: Title — gap → covered by M00N/S##/T##
### VERIFIED (skip implementation, add test if missing)
- Story X.X, X.X, X.X ...
```

### 4.4 Update downstream task plans

After writing IMPLEMENTATION-STATUS.md:

For each **VERIFIED** story: find its task plan and add at the top of Steps:
```
> **AUDIT RESULT:** Story X.X is VERIFIED (evidence: file:line). Skip implementation.
> If no test exists for this AC, add one. Otherwise mark this task done immediately.
```

For each **PARTIAL** story: confirm the task plan's gap description matches the audit finding. Update if mismatched.

For each **MISSING** story: confirm a task plan covers it. If not: create one.

### 4.5 Vision coverage cross-check

After auditing what's implemented, verify the GSD plan covers the **full original vision**:

```bash
# Count stories in source epic
grep "^### Story" _bmad-output/planning-artifacts/epics.md | wc -l

# Find stories that have no corresponding task plan
grep "^### Story" _bmad-output/planning-artifacts/epics.md | while read line; do
  story=$(echo "$line" | grep -oE 'Story [0-9]+\.[0-9]+')
  if ! grep -r "$story" .gsd/milestones/*/slices/*/tasks/ --include="*.md" -q; then
    echo "NO TASK PLAN: $story"
  fi
done
```

For every story with no task plan: it's a planning gap. Add it to the appropriate slice.

---

## Phase 5: Validate Auto-Mode Readiness

Run all checks. Fix anything that fails before handing off to `/gsd auto`.

### 5.1 Roadmap slice parser check

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
  while((m=re.exec(content))!==null) slices.push(m[2]+(m[1]==='x'?'✅':'⬜'));
  console.log(mid+': '+slices.length+' slices — '+slices.join(', '));
  if(slices.length===0) console.error('  ❌ ZERO slices parsed — fix roadmap format');
});
"
```

Every milestone must parse at least one slice. Zero means the format is wrong.

### 5.2 Task plan self-containment check

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
      const good = c.includes('## Steps') && c.includes('## Must-Haves') && c.includes('## Verification');
      if (good) ok++; else { stubs++; console.log('STUB:', path.join(dir,d.name)); }
    }
  });
}
check('.gsd/milestones');
console.log('Self-contained:', ok, '  Stubs:', stubs);
if(stubs>0) process.exit(1);
"
```

Must show `Stubs: 0`.

### 5.3 Missing milestone summary check

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
    console.log('❌ MISSING SUMMARY:', mid, '— GSD will activate this to write summary instead of moving on');
});
"
```

Must produce no output.

### 5.4 Routing simulation

```bash
node -e "
const fs = require('fs'), path = require('path');
function rmf(mid,suffix) {
  const f=path.join('.gsd/milestones',mid,mid+'-'+suffix+'.md');
  return fs.existsSync(f)?f:null;
}
function deps(mid) {
  const cf=rmf(mid,'CONTEXT'); if(!cf) return [];
  const c=fs.readFileSync(cf,'utf-8').trimStart();
  if(!c.startsWith('---')) return [];
  const rest=c.slice(c.indexOf('\n')+1);
  const end=rest.indexOf('\n---'); if(end===-1) return [];
  const out=[]; let in_=false;
  for(const l of rest.slice(0,end).split('\n')){
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
  if(!rmf(mid,'ROADMAP')){console.log(mid+': no roadmap');continue;}
  if(allDone(mid)){
    const sf=rmf(mid,'SUMMARY');
    if(!sf&&!active){console.log(mid+': ❌ no SUMMARY → GSD activates here to write it (fix this!)');active=mid;break;}
    console.log(mid+': ✅ complete'); continue;
  }
  const unmet=deps(mid).filter(d=>!done.has(d));
  if(unmet.length>0){console.log(mid+': ⏳ blocked on ['+unmet+']');continue;}
  console.log(mid+': 🔵 ACTIVE ✓'); active=mid; break;
}
console.log('\n→ GSD auto starts at:', active);
"
```

Confirm the output shows the intended milestone as ACTIVE.

### 5.5 Test-fix loop coverage check

```bash
for slice in .gsd/milestones/*/slices/*/; do
  plan="${slice}$(basename $slice)-PLAN.md"
  [ -f "$plan" ] || continue
  pending=$(grep -c "^- \[ \]" "$plan" 2>/dev/null || echo 0)
  [ "$pending" -eq 0 ] && continue
  if ! grep -qi "test.*fix.*confirm\|fix.*verify\|fix and confirm\|fix, and confirm" "$plan"; then
    echo "❌ MISSING fix-and-test task: $plan"
  fi
done
echo "Fix-and-test check complete"
```

Must produce no `❌` lines.

---

## Common Pitfalls

**Wrong completion markers** — Migration heuristics guess. Filesystem evidence doesn't. Always run A.3 ground truth verification before accepting any `[x]`/`[ ]` marker.

**Orphaned code** — Code written without GSD tracking is common. A file existing doesn't mean GSD knows about it. Run A.4 before writing any task plans for "not started" phases.

**Missing SUMMARY.md** — Any milestone with all slices `[x]` but no SUMMARY.md gets activated before your target. Always write summaries for shipped milestones.

**Stub task plans** — Auto-mode uses the task plan as its primary instructions. "See slice plan for details" gives the agent nothing. Every `T##-PLAN.md` must pass the B.7 self-containment test.

**No test-fix loop** — Every slice needs a final "Test, Fix, and Confirm" task. Implementation without a fix loop produces half-working features. Tests must be GREEN before moving to the next slice.

**Audit after implementation** — The implementation audit must be the FIRST slice of the active milestone. Running it after implementation means blind rebuilding of things already done and blind skipping of real gaps.

**File existence ≠ VERIFIED** — A 500-line file does not mean the story is implemented. Read the ACs. Check that each one is covered. Check that a test exists that would catch a regression.

**Oversized slices** — BMAD epics map to 7-10 tasks. Split before enriching. Split rule: find the natural concern or demo boundary.

**Wrong roadmap format** — Parser is strict:
`- [ ] **S01: Title** \`risk:medium\` \`depends:[]\``
Run the parser check after EVERY roadmap file you write.

**No vision coverage check** — Some BMAD stories never make it into .planning phases. Run Phase 4.5 cross-check to find planning gaps before they become shipping gaps.
