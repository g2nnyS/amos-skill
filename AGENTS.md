# Repository Guidelines

## Project Overview

`amos-skill` is an **Agent Skill package** for authoring importable `policy.yaml` governance documents for the AMOS governance console. It is a pure-documentation repository (no executable source code, no build system). The skill teaches an AI agent how to:

1. Confirm target authorization and scope before drafting
2. Research project-specific governance requirements
3. Author complete policy YAML documents with proper field semantics
4. Validate drafts via the AMOS HTTP API (`/import`, `/simulate`, `/validate`)
5. Submit proposals for human approval (agents never approve/publish)

**Primary use**: drop the entire directory into an agent-skills folder (e.g., `~/.claude/skills/amos-skill/`) and invoke via the `Skill` tool.

---

## Architecture & Data Flow

```
Agent workflow:
  1. Read SKILL.md         ← entry contract (scope, authorization, flow rules)
  2. Research project      ← read-only repo analysis
  3. Pick template         ← examples/global.yaml or examples/project.yaml
  4. Draft policy.yaml     ← follow references/authoring.md field/operator specs
  5. POST /import          ← send raw YAML to AMOS control plane
  6. POST /simulate        ← run candidate through rule engine (positive + negative tests)
  7. POST /proposals       ← propose to human for approval+publish
```

**Artifact flow**:
- `SKILL.md` → `references/authoring.md` (detailed specs) → `examples/{global,project}.yaml` (templates) → agent-authored `policy.yaml` (deliverable)
- `evals/{global-write,project-ledgers}/` → eval rubrics for behavioral grading

**Safety invariants**:
- Agents produce drafts only; approval, publication, and all edits to live `.pi/activation.json` or `.pi/amos.json` are human-only operations.
- Never fabricate verification when no AMOS service is available.
- Existing rules, activation, and injection structures must be preserved when editing existing policies.
- All operation evidence (`source` object) must be from actual scanned files or null (no fabrication).

---

## Key Directories

| Path | Purpose |
|------|---------|
| (root) | `SKILL.md` (entry), `README.md` (usage guide) |
| `references/` | `authoring.md` — full operator reference, verification scripts, checklist, machine contracts |
| `examples/` | `global.yaml`, `project.yaml` — canonical starting templates, complete examples |
| `evals/global-write/` | Eval case: authoring a new global baseline policy; prompt + graders |
| `evals/project-ledgers/` | Eval case: editing project supplementary rules; prompt + graders |

---

## Development Commands

**No in-repo build, test, or dev commands exist.**

Verification requires external services:

### Online Verification (against AMOS console)
Requires running AMOS control plane at `http://127.0.0.1:$AMOS_PORT` with `AMOS_AGENT_TOKEN` set.

```bash
# Full examples in references/authoring.md §4
export AMOS_PORT=3000
export AMOS_AGENT_TOKEN="<token>"
export PROJECT_ID="<global or UUID>"
export DRAFT='path/to/draft.yaml'

# Step 1: Import (always first)
node --input-type=module << 'EOF'
import fs from "fs";
const yaml = fs.readFileSync(process.env.DRAFT, "utf-8");
const resp = await fetch(`http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/${process.env.PROJECT_ID}/import`, {
  method: "POST",
  headers: { "Authorization": `Bearer ${process.env.AMOS_AGENT_TOKEN}`, "Content-Type": "text/yaml" },
  body: yaml
});
const { candidate } = await resp.json();
console.log(JSON.stringify(candidate, null, 2));
EOF

# Step 2: Simulate (using returned candidate)
node --input-type=module << 'EOF'
const candidate = /* from step 1 */;
const resp = await fetch(`http://127.0.0.1:${process.env.AMOS_PORT}/api/projects/${process.env.PROJECT_ID}/simulate`, {
  method: "POST",
  headers: { "Authorization": `Bearer ${process.env.AMOS_AGENT_TOKEN}`, "Content-Type": "application/json" },
  body: JSON.stringify({ candidate, action: /* test action */ })
});
console.log(await resp.text());
EOF
```

### Offline Verification (against local AMOS build)
Requires AMOS checkout built with `npm ci && npm run build` at `$AMOS_ROOT`.

```bash
# Full examples in references/authoring.md §5
export AMOS_ROOT="/path/to/amos"
node --input-type=module << 'EOF'
import { parsePolicyDocument } from "$AMOS_ROOT/dist/control-plane/document.js";
import { createRuleValidator } from "$AMOS_ROOT/dist/control-plane/compiler.js";
const draft = /* policy YAML parsed to object */;
const doc = parsePolicyDocument(draft);
const validator = createRuleValidator(doc.rules, { /* schema */ });
console.log(validator.validate(/* test action */));
EOF
```

**No `npm scripts`, `Makefile`, or `package.json` in this repo.**

---

## Code Conventions & Common Patterns

### Output Contract

Every agent-authored `policy.yaml` must be a **complete**, **single** YAML document:

```yaml
format: amos-project-policy
version: 1
activation:
  includedRuleIds: [rule-1, rule-2]
  excludedRuleIds: []
injection:
  baselineRuleIds: []
  routes: []
rules:
  - id: rule-1
    # ... (14 required fields per rules/document schema)
```

**Rules**: 
- All 14 fields required (see `references/authoring.md` field table).
- Deterministic rules: non-null `predicate` + `enforcement: runtime|ci|server`.
- Advisory rules: exactly `predicate: null` + `enforcement: advisory`.

### Predicate Syntax (AND-only list of operators)

```yaml
predicate:
  - op: op_in
    args: [file_read, file_write]
  - op: target_glob
    args: ["${SESSION_WORKSPACE}/src/**/*.ts"]
  - op: target_outside_glob
    args: ["${SESSION_WORKSPACE}/.git/**"]
  - op: url_query_key_in
    args: ["api_key"]
  - op: target_scheme_in
    args: ["http", "https"]
  - op: target_host_in
    args: ["example.com"]
  - op: target_host_not_in
    args: ["internal-only.local"]
```

**Glob semantics**:
- `*` matches within a directory (does not cross `/`).
- `**` matches across directories.
- `${SESSION_WORKSPACE}` expanded at verification time (project rules only, never global).
- Target paths always normalized to absolute paths.

### ID Discipline

- New rule IDs added to explicit `activation.includedRuleIds`.
- Deleted rule IDs purged from `excludedRuleIds`, `baselineRuleIds`, route `add`, and route `suppress`.
- Never reuse, rename, or change semantics of existing IDs.
- Project rules never "promoted" to global baseline.

### Evidence & Source Objects

For hard rules (file operations only: `file_read`, `file_write`, `file_search`, `file_list`, `file_mkdir`, `file_delete`):

```yaml
source:
  repo: "primary"
  rev: "HEAD"
  path: "docs/governance.md"
  section: "Security Policy"
  lines: [42, 50]
  quote: "All secrets must be encrypted..."
```

**Rules**:
- All fields (`repo`, `rev`, `path`, `section`, `lines`, `quote`) must be non-empty **or** the entire `source` must be `null`.
- No fabrication; all evidence from scanned files.
- Never include secrets or sensitive data in `quote`.
- Shell operations (`shell_input`, `file_stat`, `http_request`) never match hard rules; use advisory-only for those.

### Verification Flow

1. **Always `POST /import` first** — validates document structure, returns a `candidate` object.
2. **Use the returned `candidate`** in subsequent `/validate` and `/simulate` calls (raw `{yaml}` silently drops `injection`).
3. **Run ≥1 positive + ≥1 negative simulations** — test both should-match and should-not-match actions.
4. **Offline runs must note what wasn't verified** (e.g., "HTTP content routing checked offline; `target_host_in` not fully exercised without live API").

### Human Approval Boundary

Agents **never**:
- Approve or publish policies (`POST /api/proposals/:id/decision`).
- Edit `.pi/activation.json` or `.pi/amos.json` directly.
- Roll back policies.
- Modify database or service state.

Agents **only**:
- Draft and import proposals.
- Simulate and validate.
- Submit proposals with `{title, rationale, baseVersion, candidate}` for explicit user authorization.

### Language & Style

- All documentation, narratives, and comments in **Simplified Chinese** (zh-CN).
- Technical terms (operator names, field names, API paths) in English.
- Use "命中" (match/hit) and "不命中" (not match) for rule matching semantics.
- Examples include explanatory Chinese comments showing the import-first flow.

---

## Important Files

| File | Lines | Purpose |
|------|-------|---------|
| **SKILL.md** | 122 | Skill manifest (YAML frontmatter: `name`, `description`). **Entry point.** Contains: scope confirmation rules, read-only research requirements, operation-mapping table, 14 required rule fields, 7 operators, `${SESSION_WORKSPACE}` semantics, global vs. project model, human-approval boundary, full verification flow. |
| **README.md** | ~40 | Human-facing guide: project purpose, example prompts, AMOS HTTP API table, skill installation path (`~/.claude/skills/amos-skill/`). |
| **references/authoring.md** | 212 | **Authoritative reference.** Machine contracts (points to external AMOS repo: `constraints/schema.json`, `constraints/vocabulary.md`, `control-plane/document.ts`, `control-plane/compiler.ts`), operator argument tables, project-survey checklist, two complete Node.js verification scripts (online §4, offline §5), delivery checklist (§6). |
| **examples/global.yaml** | — | Complete global baseline policy example: `format`, `version`, `activation` (explicit `includedRuleIds`), `injection`, 2 canonical rules (deterministic deny + advisory). |
| **examples/project.yaml** | — | Complete project supplementary policy example: `${SESSION_WORKSPACE}` globs, `injection.routes` with `pathHint`/`add`/`suppress`/`specificity`. |
| **evals/global-write/** | — | Eval case: author a new global baseline. Contains `case.yaml` (schema_version 1.1, expected_outcome), `prompt.md` (enhanced with real project-amos repo scanning, mapping table, 10+ rules, 2+ rule simulations), and 7 LLM graders (document validity, inventory & secrets, hard-rule scope, project separation, positive/negative matching, injection consistency, hard/advisory selection). Reference: `docs/global-baseline-generation.md`. |
| **evals/project-ledgers/** | — | Eval case: author project-level supplementary rules from AGENTS.md. Contains `case.yaml` (schema_version 1.1, expected_outcome with 5-8 rules, `${SESSION_WORKSPACE}` usage), `prompt.md` (project doc scanning, mapping table, workspace-path usage), and 3 LLM graders (document validity, project-specific scope & evidence & separation from baseline, import & simulate verification & project uniqueness). Reference: `docs/project-rules-from-agents.md`. |
| **docs/global-baseline-generation.md** | — | **Complete step-by-step guide** for authoring a new global baseline from scratch: environment setup, project-amos repo scanning, operation mapping, 10+ rules with evidence, import/simulate verification, boundary report. 8 detailed sections with examples and validation scripts. |

| **docs/project-rules-from-agents.md** | — | **Complete step-by-step guide** for authoring project-level supplementary rules from AGENTS.md: project context extraction, security constraints & engineering conventions, `${SESSION_WORKSPACE}` usage, separation from global baseline, mapping table, policy.yaml authoring, import & simulate verification, boundary report. 8 detailed sections with examples and validation scripts. |
| **docs/multi-agent-governance.md** | — | **Advanced governance for multi-agent systems**: distinguishing benign operations from malicious ones using context, operation sequences, and trust models. Covers layered rules (hard-deny, confirmation, advisory), real-world scenarios, pattern analysis, inter-agent trust, and integration with AMOS. 9 sections with practical examples and future roadmap. |
---

## Runtime & Tooling Preferences

### No In-Repo Dependencies

This repository contains **no executable code**, `package.json`, `Makefile`, or build system.

### External Requirements

**Online Verification**:
- **Node.js 26+** (uses native `fetch`).
- Running AMOS control-plane service at `http://127.0.0.1:$AMOS_PORT`.
- Environment: `AMOS_PORT`, `AMOS_AGENT_TOKEN`, `PROJECT_ID` (literal `global` or UUID).

**Offline Verification**:
- Separate AMOS repository checked out at `$AMOS_ROOT`.
- AMOS built: `npm ci && npm run build` → `dist/control-plane/document.js`, `dist/control-plane/compiler.js`, `constraints/schema.json`.
- Imports: `parsePolicyDocument`, `createRuleValidator`, `validateAndCompileCandidate`.

### Skill Consumption

- Directory dropped into agent skills folder (e.g., Claude Code's `~/.claude/skills/amos-skill/`).
- Invoked via `Skill` tool or agent-native skill loading.
- Entry point: `SKILL.md` frontmatter (`name`, `description`).

### No Linting, Formatting, or CI/CD

- No `.github/workflows/`, pre-commit hooks, or automated checks.
- No eslint, prettier, or similar tools.
- Quality gate is human review + rubric-based eval grading.

---

## Testing & QA

### No Automated Test Framework

This repository has **no test runner, assertions, or fixtures** in the traditional sense. Testing is **qualitative and external**.

### Evaluation Cases

Two structured eval scenarios (not executable harnesses):

| Path | Purpose | Schema | Graders |
|------|---------|--------|---------|
| `evals/global-write/` | Author a new global baseline policy | `case.yaml` schema_version 1.1 | 7 LLM graders: document validity, inventory, hard-rule scope, project separation, matching, injection, selection |
| `evals/project-ledgers/` | Edit existing project supplementary rules | `case.yaml` schema_version 1.1 | 3 LLM graders: document validity, scope coverage, advisory-only rules |

Each eval case contains:
- **`case.yaml`**: scenario metadata (`name`, `expected_outcome` in Chinese).
- **`prompt.md`**: agent prompt (frontmatter: `tags`, `max_turns`, `timeout_seconds`, `allowed_tools`; body: task description).
- **`graders/*.md`**: rubric-based LLM graders (`type: llm, focus: trace, weight: 1`), one per evaluation concern.

### Verification Model

**Online (against live AMOS service)**:
1. `POST /import` with raw YAML → returns `candidate`.
2. `POST /simulate` with `candidate` and test action (positive case) → verify rule matches expected.
3. `POST /simulate` with `candidate` and negative test action → verify rule does not match.
4. Inspect any error responses.

**Offline (against local AMOS build)**:
1. Parse policy YAML with `parsePolicyDocument`.
2. Create validator with `createRuleValidator`.
3. Run test actions through validator.
4. Document any operations not fully verified offline.

**Human Checklist** (`references/authoring.md` §6):
- [ ] All 14 rule fields present.
- [ ] No `...` placeholders or TODOs.
- [ ] `${SESSION_WORKSPACE}` only in project rules.
- [ ] ID discipline: new IDs added to `includedRuleIds`, deleted IDs purged from all references.
- [ ] Positive + negative simulations completed.
- [ ] No secrets in `source.quote`.
- [ ] Ready for proposal submission.

### No Coverage Expectations

- No code coverage metrics.
- Quality measured by rubric grading and human spot-checks.
- Evaluation focuses on: document validity, scope matching, advisory-rule correctness.
