---
name: gsd-onboard
description: Migrate any project's existing planning artifacts into GSD so that `/gsd auto` starts at exactly the right milestone, every task plan is self-contained, the full vision is covered, and the codebase is verified against the plan before launch.
---

You are onboarding a project into GSD. Your job: produce a `.gsd/` directory where `/gsd auto` activates at precisely the right milestone with accurate completion markers, self-contained task plans, and zero gaps against the original vision.

Follow every phase in order. Do not skip any step.

---

## Phase 0: Understand the Project

Before touching any files, answer these four questions:

**1. What planning artifacts exist?**
```bash
ls -la .planning/ 2>/dev/null && echo "HAS .planning"
ls -la .gsd/ 2>/dev/null && echo "HAS .gsd"
find . -maxdepth 4 \( \
  -name "epics*.md" -o -name "*.epic.md" \
  -o -name "sprint*.yaml" -o -name "sprint*.json" \
  -o -name "*roadmap*.md" -o -name "prd.md" \
  -o -name "BACKLOG.md" -o -name "TODO.md" \
  -o -name "*.stories.md" \
\) 2>/dev/null | grep -v node_modules | grep -v ".gsd"
ls ~/.gsd/agent/extensions/gsd/migrate/ 2>/dev/null && echo "gsd-migrate INSTALLED"
```

**2. What code already exists?**
```bash
# Get a feel for the codebase size and structure
find . -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.py" -o -name "*.go" \
  -o -name "*.js" -o -name "*.jsx" -o -name "*.rs" \) \
  2>/dev/null | grep -v node_modules | grep -v ".git" | wc -l
ls src/ app/ lib/ packages/ 2>/dev/null
```

**3. What test runner does this project use?**
```bash
cat package.json 2>/dev/null | grep -E '"test"|"vitest"|"jest"|"mocha"'
cat pyproject.toml setup.cfg 2>/dev/null | grep -E "pytest|unittest"
cat go.mod 2>/dev/null | head -3
cat Makefile 2>/dev/null | grep -E "^test"
```
Record the exact test command — every "Test, Fix, and Confirm" task will use it.

**4. What is the project's type check / lint command?**
```bash
cat package.json 2>/dev/null | grep -E '"typecheck"|"lint"|"check"'
cat Makefile 2>/dev/null | grep -E "^lint|^check|^type"
```
Record this too.

**5. What skills, agents, and tool integrations exist?**
```bash
# Installed GSD skills
ls ~/.gsd/agent/skills/ 2>/dev/null
# Project-local agents/skills (BMAD, custom)
ls ~/.agents/skills/ 2>/dev/null
find . -maxdepth 4 \( -name "agent-manifest.csv" -o -name "agents" -type d \) \
  2>/dev/null | grep -v node_modules | grep -v ".git"
# MCP servers or tool bridges
find . -maxdepth 4 -name "*.ts" -path "*/mcp/*" 2>/dev/null | grep -v node_modules
find . -maxdepth 4 -name "*.py" -path "*/mcp/*" 2>/dev/null | grep -v node_modules
# GSD preferences (global and project)
cat ~/.gsd/preferences.md 2>/dev/null
cat .gsd/preferences.md 2>/dev/null
```
Record:
- Which skills are installed and relevant to this project
- Whether an agent manifest exists (for party mode / multi-agent workflows)
- What MCP servers or external tool integrations exist
- Current GSD preferences (if any)

These will be preserved or configured in Phase 3.6.

---

Based on your findings, take the matching path:

| Situation | Path |
|---|---|
| `.planning/` exists + `gsd-migrate` installed | **Path A** — automated migration |
| `.planning/` exists + NOT installed | Install: `curl -fsSL https://raw.githubusercontent.com/jonathancostin/gsd-migrate/main/install.sh \| bash`, then Path A |
| Any planning docs exist (epics, PRD, backlog, sprints, issues) | **Path B** — manual migration |
| `.gsd/` already exists but is messy, wrong, or partial | **Path C** — audit and fix |
| No planning docs and no `.gsd/` | **Path D** — bootstrap from scratch |

---

## Path A: Automated .planning Migration

### A.1 Run the migration

```bash
/gsd migrate
```

Review preview stats before confirming. Verify milestone count, slice count, task count, and completion percentages look plausible before writing.

### A.2 Validate roadmap format immediately

Run this right after migration writes files — before doing anything else:

```bash
node -e "
const fs = require('fs'), path = require('path');
const re = /^- \[(x| )\] \*\*(\w+): (.+?)\*\* \x60risk:(\w+)\x60 \x60depends:\[([^\]]*)\]\x60/gm;
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d=>{
  const mid = d.name.match(/^(M\d+)/)?.[1]; if(!mid) return;
  const f = path.join('.gsd/milestones',d.name,mid+'-ROADMAP.md');
  if(!fs.existsSync(f)) return;
  const content = fs.readFileSync(f,'utf-8');
  let count=0, m;
  while((m=re.exec(content))!==null){count++; console.log('OK:',m[2],m[1]==='x'?'[done]':'[pending]');}
  if(count===0) console.error('❌ ZERO slices parsed in',mid,'— fix roadmap format before continuing');
  else console.log(mid+': '+count+' slices OK');
});
"
```

If any milestone shows zero slices: stop and fix the roadmap format before continuing. See Common Pitfalls for the required format.

### A.3 Ground truth verification — filesystem beats migration heuristics

**Do not trust the migration output's `[x]`/`[ ]` markers.** The migration tool guesses completion from heuristics. Ground truth comes from the filesystem.

```bash
for phase in .planning/phases/*/; do
  plans=$(ls "$phase"*-PLAN.md 2>/dev/null | wc -l)
  summaries=$(ls "$phase"*-SUMMARY.md 2>/dev/null | wc -l)
  name=$(basename "$phase")
  if [ "$summaries" -eq "$plans" ] && [ "$plans" -gt 0 ]; then
    echo "DONE    $name ($summaries/$plans summaries)"
  elif [ "$summaries" -gt 0 ]; then
    echo "PARTIAL $name ($summaries/$plans summaries)"
  else
    echo "PENDING $name (0/$plans summaries)"
  fi
done
```

Rules:
- **DONE** = every plan file has a matching summary file
- **PARTIAL** = some plans have summaries, others don't
- **PENDING** = no summary files exist

Cross-check against the migrated roadmap `[x]`/`[ ]` markers. Fix any that disagree with the filesystem evidence before continuing.

### A.4 Detect orphaned code

For every PENDING or PARTIAL phase: check whether implementing code already exists before writing task plans from scratch.

```bash
# Read each pending phase's plan files to find expected output filenames
# then search the actual codebase for those files
for phase in .planning/phases/*/; do
  summaries=$(ls "$phase"*-SUMMARY.md 2>/dev/null | wc -l)
  plans=$(ls "$phase"*-PLAN.md 2>/dev/null | wc -l)
  [ "$summaries" -lt "$plans" ] || continue
  echo "=== $(basename $phase) ==="
  # Extract filenames mentioned in plan files and search for them
  grep -h "\.(ts\|tsx\|py\|go\|js\|jsx\|rs\|rb)" "$phase"*.md 2>/dev/null \
    | grep -oE '[a-zA-Z0-9_-]+\.(ts|tsx|py|go|js|jsx|rs|rb)' | sort -u \
    | while read fname; do
        found=$(find . -name "$fname" -not -path "*/node_modules/*" -not -path "*/.git/*" 2>/dev/null | head -1)
        [ -n "$found" ] && echo "  EXISTS: $found"
      done
done
```

For each file found in a "pending" phase:
- Read the requirements/ACs for that phase
- Read the found file — does it implement those requirements?
- **Fully implemented + tests exist** → write a SUMMARY.md marking it done
- **Partially implemented** → write task plan covering only remaining requirements
- **Skeleton/stub** → write a full task plan from scratch

### A.5 Review checklist

1. All roadmaps parse correctly (no zero-slice milestones)
2. `[x]`/`[ ]` markers match filesystem ground truth from A.3
3. Orphaned code handled (A.4)
4. Read 2–3 task plans — confirm they are self-contained (see Phase 3.3)
5. PROJECT.md has the real project description (not boilerplate)

Proceed to Phase 3.

---

## Path B: Manual Migration from Any Planning Format

### B.1 Read the source planning artifacts

Read whatever planning documents exist. Common formats:

```bash
# BMAD
cat _bmad-output/planning-artifacts/epics.md 2>/dev/null
cat _bmad-output/planning-artifacts/prd.md 2>/dev/null

# Generic
cat BACKLOG.md ROADMAP.md TODO.md 2>/dev/null
find . -name "*.stories.md" -o -name "*epic*.md" 2>/dev/null | xargs cat

# GitHub issues export (if present)
find . -name "issues*.json" -o -name "issues*.csv" 2>/dev/null | head -3

# PRD / spec docs
find . -maxdepth 3 -name "*.md" | xargs grep -l "requirements\|acceptance criteria\|user story" 2>/dev/null | head -10
```

After reading, record:
- **Project name** and one-sentence description
- **Tech stack** — language, framework, key libraries, hard constraints
- **Done work** — features confirmed shipped (has code + passing tests)
- **In-progress work** — features partially implemented
- **Backlog** — features not yet started
- **Key decisions** — architectural choices already made

### B.2 Establish ground truth from the codebase

Before designing any milestone structure, verify what's actually built. For each claimed "done" feature:

```bash
# Find the implementing code
rg "featureName|ComponentName|/api/route" src/ --include="*.ts" --include="*.py" --include="*.go" -l

# Confirm tests exist
rg "featureName|test.*feature" tests/ src/ --include="*.test.*" --include="*_test.*" --include="test_*.py" -l
```

A feature is only **done** if: code exists AND the requirements are fully covered AND tests exist that would catch a regression.

### B.3 Design the milestone structure

Read the GSD templates first:
```bash
cat ~/.gsd/agent/extensions/gsd/templates/roadmap.md
cat ~/.gsd/agent/extensions/gsd/templates/plan.md
cat ~/.gsd/agent/extensions/gsd/templates/task-plan.md
```

Structure rules:
- **One milestone** = one major project phase (e.g. MVP Foundation, Backend API, Intelligence Layer, Launch)
- **Completed work** → done milestones with `[x]` slices + `MILESTONE-SUMMARY.md`
- **Active work** → current milestone with pending slices
- **Max 5 active tasks per slice** — split larger chunks (see Phase 3.2)

Map source artifacts to GSD:

| Source format | GSD equivalent |
|---|---|
| Epic / large feature (3-5 stories) | 1–2 Slices |
| Epic / large feature (6+ stories) | Multiple slices — split at natural concern boundary |
| Story / issue / ticket | Task |
| Done sprint / shipped release | Done slices `[x]` with SUMMARY.md |
| ADR / architecture decision | Row in DECISIONS.md |
| PRD requirement / acceptance criterion | Row in REQUIREMENTS.md |

### B.4 Write core GSD docs

```bash
cat ~/.gsd/agent/extensions/gsd/templates/project.md
cat ~/.gsd/agent/extensions/gsd/templates/decisions.md
```

- **PROJECT.md** — current state only. Tech stack with hard constraints explicit. Do not write aspirational content — only what exists right now.
- **DECISIONS.md** — append-only table: `| # | When | Scope | Decision | Choice | Rationale | Revisable? |`. Extract from ADRs or planning docs.

### B.5 Write milestone roadmaps

**Parser-critical slice format — must be exact or GSD cannot read it:**
```
- [x] **S01: Title** `risk:medium` `depends:[]`
- [ ] **S02: Title** `risk:high` `depends:[S01]`
```

Rules:
- `[x]` = done, `[ ]` = pending
- Title must be bold with colon: `**S01: Title**`
- Both backtick-wrapped tags required: `` `risk:low|medium|high` `` and `` `depends:[S01,S02]` `` or `` `depends:[]` ``

**After writing each roadmap file, immediately validate:**
```bash
node -e "
const fs = require('fs');
const re = /^- \[(x| )\] \*\*(\w+): (.+?)\*\* \x60risk:(\w+)\x60 \x60depends:\[([^\]]*)\]\x60/gm;
const content = fs.readFileSync('.gsd/milestones/M001/M001-ROADMAP.md','utf-8');
let count=0, m;
while((m=re.exec(content))!==null){count++;console.log('OK:',m[2]);}
if(count===0) throw new Error('ZERO slices parsed — fix format before writing slice files');
console.log(count,'slices parsed OK');
"
```

If count is 0: fix the roadmap before writing any slice files.

### B.6 Write slice plans

Required sections: Goal, Demo, Must-Haves, Proof Level, Verification, Observability / Diagnostics, Integration Closure, Tasks, Files Likely Touched.

**Parser-critical task format:**
```
- [ ] **T01: Title** `est:2h`
  - Why: reason this task exists
  - Files: `path/to/file.ext`
  - Do: specific steps to take
  - Verify: exact command or action to confirm it worked
  - Done when: observable state that proves completion
```

**Every implementation slice must end with a "Test, Fix, and Confirm" task** (see Phase 3.3 for the template).

### B.7 Write task plan files

**Before writing each `T##-PLAN.md`, verify it will have all of:**
- [ ] `## Description` — what this builds and why, in plain language
- [ ] `## Steps` — numbered, specific, executable steps (not "implement X" — say exactly HOW)
- [ ] `## Must-Haves` — checkbox list of acceptance criteria from the source requirements
- [ ] `## Verification` — exact commands to run or exact browser/UI actions to take
- [ ] `## Inputs` — specific file paths this task reads from prior work
- [ ] `## Expected Output` — specific file paths this task creates or modifies

**Self-containment test:** Hide the slice plan. Can GSD auto execute this task using only the task plan and the repo? If no → rewrite.

**Never write:** "see slice plan for details", "refer to requirements doc", "implement as discussed". Every context the agent needs must be in the task plan file.

When referencing source requirements:
- Quote the requirement text or acceptance criterion directly — do not just reference a line number
- Include the source location as a navigation hint, not as a replacement for the content

---

## Path C: Audit and Fix Existing .gsd

Use this when `.gsd/` already exists but is wrong, messy, or partially complete.

1. Run roadmap format validation (B.5 script) for every milestone — note which fail
2. Run ground truth verification (A.3) — fix any wrong `[x]`/`[ ]` markers
3. Run the missing-summary check (Phase 3.1 script) — fix any gaps
4. Run the self-containment check (Phase 3.3 script) — list all stubs
5. Check task counts per pending slice: `grep -c "^- \[ \]" .gsd/milestones/*/slices/*/S*-PLAN.md` — split any with > 5
6. Note all issues, then proceed through Phase 3 to fix each one

---

## Path D: Bootstrap from Scratch

Use this when there are no planning docs and no `.gsd/`.

1. Read the README, any spec docs, and the existing codebase to understand what exists and what's intended
2. Identify what's already built (has code + tests), what's in-progress, and what's backlog
3. Define milestones based on major deliverable phases
4. Proceed from B.3 onward — the planning source is your understanding of the project

---

## Phase 3: Hardening (ALL PATHS)

### 3.1 Write SUMMARY.md for every completed milestone

Any milestone where all slices are `[x]` but no `MILESTONE-SUMMARY.md` exists gets activated as "completing-milestone" — GSD asks the agent to write the summary instead of moving to the next milestone.

Find affected milestones:
```bash
node -e "
const fs = require('fs'), path = require('path');
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d => {
  const mid = d.name.match(/^(M\d+)/)?.[1]; if(!mid) return;
  const rp = path.join('.gsd/milestones',d.name,mid+'-ROADMAP.md');
  const sp = path.join('.gsd/milestones',d.name,mid+'-SUMMARY.md');
  if(!fs.existsSync(rp)) return;
  const slices = fs.readFileSync(rp,'utf-8').match(/^- \[(x| )\] \*\*\w+:/gm)||[];
  const allDone = slices.length>0 && slices.every(s=>s.includes('[x]'));
  if(allDone && !fs.existsSync(sp))
    console.log('❌ NEEDS SUMMARY:', mid);
});
"
```

For each: read the template and write a real summary:
```bash
cat ~/.gsd/agent/extensions/gsd/templates/milestone-summary.md
```

Required content: What was built (cross-slice narrative), how it was verified, and what the next milestone needs to know (what's fragile, what the authoritative diagnostics are).

### 3.2 Detect and split oversized slices

```bash
node -e "
const fs = require('fs'), path = require('path');
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(mid => {
  const slicesDir = path.join('.gsd/milestones',mid.name,'slices');
  if(!fs.existsSync(slicesDir)) return;
  fs.readdirSync(slicesDir,{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(sid => {
    const plan = path.join(slicesDir,sid.name,sid.name+'-PLAN.md');
    if(!fs.existsSync(plan)) return;
    const active = (fs.readFileSync(plan,'utf-8').match(/^- \[ \] \*\*T\d+/gm)||[]).length;
    if(active > 5) console.log('OVERSIZED:',mid.name+'/'+sid.name,'—',active,'active tasks');
  });
});
"
```

**Split before enriching task plans** — enrichment must write to final file locations.

To split: find the natural concern boundary or demo cutoff, create a new slice directory, move task plan files past the boundary, update both slice plan task lists, add the new slice to the milestone roadmap.

### 3.3 Add a "Test, Fix, and Confirm" task to every pending slice

Every implementation slice needs a final task that runs the full test suite and fixes every failure before the slice is considered done. Check for missing ones:

```bash
for slice in .gsd/milestones/*/slices/*/; do
  plan="${slice}$(basename $slice)-PLAN.md"
  [ -f "$plan" ] || continue
  pending=$(grep -c "^- \[ \]" "$plan" 2>/dev/null || echo 0)
  [ "$pending" -eq 0 ] && continue
  grep -qi "test.*fix\|fix.*confirm\|verify.*fix" "$plan" || echo "MISSING fix task: $plan"
done
```

For each slice missing it, add a final task using this template (adapt commands to the project's test runner):

```markdown
- [ ] **T##: Test, Fix, and Confirm** `est:2h`
  - Why: No slice is done until the full test suite passes and every surface works
  - Files: All files modified in this slice
  - Do: |
      Run the project test suite for modules touched in this slice.
      Read every failure carefully — fix the root cause, not the symptom.
      Re-run until all tests pass.
      Run the project's type check / lint command.
      Fix every error. Re-run until clean.
      For UI slices: open the browser and exercise every surface — zero console errors.
      Run the full project test suite to confirm no regressions elsewhere.
      Only then mark this task done.
  - Verify: [project test command] exits 0; [project typecheck command] exits 0
  - Done when: Zero failures anywhere in the test suite; all UI surfaces verified if applicable
```

Replace `[project test command]` and `[project typecheck command]` with the actual commands discovered in Phase 0.

### 3.4 Enforce task plan self-containment

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

### 3.5 Set depends_on routing

GSD activates the **first incomplete milestone with no unmet dependencies**. Use `depends_on` in `CONTEXT.md` to control order:

```yaml
---
depends_on:
  - M001
---
# rest of CONTEXT.md
```

Format rules: YAML frontmatter at top of file, 2-space indent for list items, milestone IDs uppercase (M001 not m001).

### 3.6 Preserve skills, agents, and tool integrations

If Phase 0 discovered installed skills, agent manifests, or MCP/tool integrations, make sure GSD auto can use them.

**Write or update `.gsd/preferences.md`:**

```bash
cat ~/.gsd/agent/extensions/gsd/docs/preferences-reference.md
```

**Important: Use absolute paths for skills outside `~/.gsd/agent/skills/`.** GSD's skill resolver only searches `~/.gsd/agent/skills/` and `.pi/agent/skills/` for bare names. Skills installed elsewhere (e.g. `~/.agents/skills/`) must use full `~/` paths or they'll show as "⚠ not found":

```yaml
# ❌ Won't resolve — bare name not in ~/.gsd/agent/skills/
prefer_skills:
  - bmad-party-mode

# ✅ Will resolve — absolute path with ~ expansion
prefer_skills:
  - ~/.agents/skills/bmad-party-mode
```

Configure preferences based on what exists:

1. **Skills** — Add relevant skills to `prefer_skills` or `skill_rules`:
   ```yaml
   prefer_skills:
     - ~/.agents/skills/bmad-party-mode
     - ~/.agents/skills/bmad-code-review
   skill_rules:
     - when: designing architecture or choosing between implementation approaches
       use:
         - ~/.agents/skills/bmad-party-mode
     - when: reviewing completed code
       use:
         - ~/.agents/skills/bmad-code-review
   ```

2. **Multi-agent / party mode auto-approve** — Party mode is interactive by default (it waits for user input). During GSD auto, there's no user. Add this custom instruction so party mode runs as non-interactive deliberation instead of blocking:
   ```yaml
   custom_instructions:
     - "PARTY MODE AUTO-APPROVE RULE: During GSD auto execution (no user present), run party mode as non-interactive deliberation. Load the agent manifest, generate 2-3 relevant agent perspectives on the approach, synthesize the consensus into a concrete decision, then continue implementing immediately — do NOT wait for user input. Document the deliberation and decision in the task summary under a '## Party Mode Deliberation' section. The user trusts the team's consensus and wants auto-mode to keep moving."
   ```

   During interactive sessions (user present), party mode runs normally with full conversation flow.

3. **Agent manifests** — If an agent manifest exists (e.g. `_bmad/_config/agent-manifest.csv`), tell the agent where to find it:
   ```yaml
   custom_instructions:
     - "Agent manifest at _bmad/_config/agent-manifest.csv — load for party mode deliberation"
   ```

4. **MCP servers / tool integrations** — Document available integrations so GSD auto knows what tools are available:
   ```yaml
   custom_instructions:
     - "MCP servers available: [list servers]. Initialized via [path]."
   ```

5. **Skill discovery** — Set to `auto` if you want GSD to find and apply skills without prompting:
   ```yaml
   skill_discovery: auto
   ```

**Do NOT add party mode annotations to task plans.** The preferences mechanism handles skill routing. Task plans must stay self-contained execution specs — adding "activate party mode" annotations violates self-containment and wastes tokens on tasks where the preferences already route correctly.

---

## Phase 4: Implementation Audit

Before any new implementation begins, produce an `IMPLEMENTATION-STATUS.md` that maps every source requirement against the actual codebase. This becomes the ground truth all downstream task plans reference — skipping VERIFIED items, completing PARTIAL ones, building MISSING ones.

### 4.1 Structure the audit

Add a dedicated audit slice as the **first pending slice** of the active milestone. No implementation task should begin until this slice is complete.

Split the audit across multiple tasks based on the number of source requirements:
- **< 20 requirements**: one audit task covering everything
- **20-50 requirements**: two tasks splitting the requirements by epic/area
- **50+ requirements**: three tasks splitting by thirds

Final task in the audit slice always: consolidate working notes into `IMPLEMENTATION-STATUS.md` and update all downstream task plans.

### 4.2 How to audit each requirement

For each requirement/story/ticket, follow this process:

1. Read the requirement's **acceptance criteria** from the source document
2. Identify the **primary artifact**: the function, file, API route, or UI component that directly implements the core AC
3. Search for it in the codebase:
   ```bash
   rg "FunctionName|ComponentName|/api/route-path" src/ -l
   # adapt file extensions to the project's language
   ```
4. If found: **read the file** — confirm it covers each AC fully (file existence ≠ implemented)
5. Check it's **wired**: is it called/imported from the main execution path, not just sitting unused?
6. Check a **test exists** that would fail if this requirement were removed:
   ```bash
   rg "FunctionName|requirement-description" tests/ -l
   ```
7. Assign status:
   - **VERIFIED**: code found + all ACs covered + wired into main path + test exists
   - **PARTIAL**: code exists but one or more ACs missing, OR not wired, OR no test
   - **MISSING**: no implementing code found at all

Assign severity to gaps:
- **HIGH**: blocks a user-visible feature or core demo scenario
- **MEDIUM**: feature works but with degraded behavior or missing edge case
- **LOW**: observability, logging, or cosmetic gap

### 4.3 IMPLEMENTATION-STATUS.md format

```markdown
# [Project Name] — Implementation Status

**Audited:** YYYY-MM-DD
**Source:** [where requirements came from — e.g. "epics.md", "GitHub issues", "PRD v2"]
**Total requirements:** N
**VERIFIED:** N | **PARTIAL:** N | **MISSING:** N

## [Epic / Area 1 Name]

| Req | Title | Status | Evidence | Gap |
|-----|-------|--------|----------|-----|
| 1.1 | User can log in | VERIFIED | src/auth/login.ts:L23 + auth.test.ts:L15 | none |
| 1.2 | Session persists across refresh | PARTIAL | src/auth/session.ts exists, refresh not tested | Missing refresh test |
| 1.3 | Password reset email | MISSING | rg finds no password reset handler | Full implementation needed |

## [Epic / Area 2 Name]
[continue for all areas]

## Summary

### MISSING — build from scratch
- Req 1.3: Password reset email → S03/T02

### PARTIAL — complete these
- Req 1.2: Session persists → S02/T03 (add refresh test)

### VERIFIED — skip implementation
- Req 1.1, 2.1, 2.3, 3.1 ...
```

### 4.4 Update downstream task plans

After writing `IMPLEMENTATION-STATUS.md`:

For each **VERIFIED** requirement: find its task plan and prepend to the Steps section:
```
> **AUDIT RESULT:** [Req ID] is VERIFIED (see IMPLEMENTATION-STATUS.md).
> Implementation is complete. If a test is missing, add one. Otherwise mark this task done.
```

For each **PARTIAL** requirement: confirm the task plan's gap description matches the audit finding — update if mismatched.

For each **MISSING** requirement: confirm a task plan exists that covers it — create one if not.

### 4.5 Vision coverage check

Confirm every source requirement has a corresponding GSD task plan:

```bash
# Adapt the grep pattern to match how requirements are identified in your source doc
# e.g. "Story X.X", "Issue #N", "REQ-N", "FR-N"
grep -oE "Story [0-9]+\.[0-9]+" path/to/source-requirements.md | while read req; do
  grep -r "$req" .gsd/milestones/*/slices/*/tasks/ --include="*.md" -q \
    || echo "NO TASK PLAN: $req"
done
```

Adapt the pattern to whatever format your project uses for requirement IDs. For every requirement with no task plan: add it to the appropriate slice.

---

## Phase 5: Validate Auto-Mode Readiness

Run every check. Fix anything that fails before handing off to `/gsd auto`.

### 5.1 Roadmap parser check

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
  while((m=re.exec(content))!==null) slices.push(m[2]+(m[1]==='x'?' ✅':' ⬜'));
  if(slices.length===0) console.error('❌',mid,'— zero slices parsed, fix roadmap format');
  else console.log('✅',mid+':',slices.join(', '));
});
"
```

### 5.2 Self-containment check

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
      if (good) ok++; else { stubs++; console.log('❌ STUB:', d.name, 'in', dir); }
    }
  });
}
check('.gsd/milestones');
console.log(ok,'self-contained,', stubs,'stubs');
if(stubs>0) console.error('Fix all stubs before running /gsd auto');
"
```

### 5.3 Missing summary check

```bash
node -e "
const fs = require('fs'), path = require('path');
let ok = true;
fs.readdirSync('.gsd/milestones',{withFileTypes:true}).filter(d=>d.isDirectory()).forEach(d=>{
  const mid = d.name.match(/^(M\d+)/)?.[1]; if(!mid) return;
  const rp = path.join('.gsd/milestones',d.name,mid+'-ROADMAP.md');
  const sp = path.join('.gsd/milestones',d.name,mid+'-SUMMARY.md');
  if(!fs.existsSync(rp)) return;
  const slices = fs.readFileSync(rp,'utf-8').match(/^- \[(x| )\] \*\*\w+:/gm)||[];
  const allDone = slices.length>0 && slices.every(s=>s.includes('[x]'));
  if(allDone && !fs.existsSync(sp)){
    console.error('❌',mid,'— all slices done but no SUMMARY.md — GSD will activate this to write it');
    ok = false;
  }
});
if(ok) console.log('✅ All complete milestones have summaries');
"
```

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
  if(!rmf(mid,'ROADMAP')){console.log(mid+': no roadmap — skipped');continue;}
  if(allDone(mid)){
    if(!rmf(mid,'SUMMARY')&&!active){
      console.error('❌',mid,'— complete but no SUMMARY — GSD activates here (fix!)');active=mid;break;
    }
    console.log('✅',mid,'complete'); continue;
  }
  const unmet=deps(mid).filter(d=>!done.has(d));
  if(unmet.length){console.log('⏳',mid,'blocked on ['+unmet+']');continue;}
  console.log('🔵',mid,'ACTIVE'); active=mid; break;
}
console.log('\n→ /gsd auto starts at:', active||'none found — check structure');
"
```

Confirm the output shows the milestone you intended as ACTIVE.

### 5.5 Test-fix loop coverage

```bash
missing=0
for slice in .gsd/milestones/*/slices/*/; do
  plan="${slice}$(basename $slice)-PLAN.md"
  [ -f "$plan" ] || continue
  pending=$(grep -c "^- \[ \]" "$plan" 2>/dev/null || echo 0)
  [ "$pending" -eq 0 ] && continue
  grep -qi "test.*fix\|fix.*confirm\|verify.*fix" "$plan" || { echo "❌ No fix task: $plan"; missing=$((missing+1)); }
done
[ "$missing" -eq 0 ] && echo "✅ All pending slices have fix tasks" || echo "$missing slices missing fix tasks"
```

---

## Common Pitfalls

**Wrong completion markers** — Migration heuristics guess completion. Filesystem summary-file counts don't. Always run A.3 before accepting any `[x]`/`[ ]` marker.

**Orphaned code** — Code written without GSD tracking is extremely common (especially when sessions end mid-task). A pending phase in your plan might be 80% implemented. Run A.4 before writing task plans for "not started" work.

**Missing SUMMARY.md on a complete milestone** — GSD will activate that milestone to write its summary instead of moving to the next one. Run check 5.3 before every handoff to `/gsd auto`.

**Stub task plans** — Auto-mode reads the task plan as its primary instructions in a fresh context. "See slice plan for details" or "implement as described in the requirements" gives the agent nothing to work with. Every `T##-PLAN.md` must pass the B.7 self-containment test.

**No test-fix loop on implementation slices** — Without a final "Test, Fix, and Confirm" task, GSD auto moves to the next slice the moment it writes the last file — even if tests are broken. The fix loop is not optional.

**Running the audit after implementation** — The implementation audit must be the first slice of the active milestone. Running it after means GSD auto may rebuild things that already exist and skip genuine gaps.

**File existence ≠ implemented** — A 500-line file does not mean the requirement is met. A function stub is not an implementation. Read the acceptance criteria. Read the file. Check that each AC is covered. Check that a test exists.

**Oversized slices** — Planning tools (BMAD, Jira, Notion) produce epics with 7-15 items. GSD works best with max 5 active tasks per slice. Split before writing task plan files — so enrichment lands in final locations.

**Wrong roadmap format** — The parser is strict. The required format is:
```
- [ ] **S01: Title** `risk:medium` `depends:[]`
```
Run the parser check (B.5 / 5.1) after writing every roadmap file, not just at the end.

**Vision coverage gaps** — Some requirements never make it from planning docs into `.planning` phases. Run Phase 4.5 cross-check to find requirements with no task plan before declaring the migration done.

**Tech-stack assumptions in task plans** — If this is a Python project, don't write `pnpm vitest run`. Discover the test runner in Phase 0 and use it consistently everywhere.
