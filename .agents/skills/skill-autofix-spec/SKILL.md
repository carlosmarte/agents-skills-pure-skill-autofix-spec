---
name: skill-autofix-spec
description: Refactor a target SKILL directory to comply with the AGENT SPEC standard. Audits and fixes YAML frontmatter (name, description, tier, optional fields), enforces the `<tier>-<slug>` directory-name convention, modularizes bloated SKILL.md bodies into scripts/, references/, and assets/ subdirectories for progressive disclosure, enforces the 500-line body limit, and verifies relative-path references. Trigger when the user asks to "refactor this skill to spec", "autofix a skill", "audit a skill against AGENT SPEC", "fix skill frontmatter", "bring a skill into compliance", or provides a path to a skill directory that needs to conform to the spec. Operates by reading and editing files directly — no bundled scripts.
allowed-tools: Bash,Read,Write,Edit,Glob,Grep
argument-hint: "<path-to-skill-directory>"
---

# skill-autofix-spec

Refactor a single SKILL directory in place so that it conforms to the AGENT SPEC standard for skills (see the inline reference in the appendix below). Operate as an agent-driven, dry-run-first audit: survey → propose → apply → verify.

## When to use

- User points at a skill directory and asks to "refactor", "autofix", "fix", "audit", or "bring to spec".
- User mentions YAML frontmatter problems, directory-name mismatches, bloated SKILL.md, or missing required fields (`name`, `description`, `tier`).
- User wants progressive-disclosure refactoring — moving bulk content out of SKILL.md into `scripts/`, `references/`, or `assets/`.

Do NOT use to **create** new skills from scratch (use `/skill-create`), to refactor unrelated documentation, or to mass-rewrite multiple skills in a sweep without explicit confirmation.

## Input

Required: an absolute or repo-relative path to a single skill directory. If the user did not provide one, ask. Accept either:

- A directory containing `SKILL.md` (treat as the skill root).
- A parent directory listing multiple skill folders — in that case, ask the user which one to refactor (this skill refactors one at a time).

## Operating principles

1. **Dry-run first.** Always survey the target and emit a written diff plan before any edit. Get user confirmation before any destructive action (directory rename, file move, content deletion, > 50-line content extraction).
2. **Non-destructive by default.** Prefer Edit over Write. Never delete content — when extracting bulk material from SKILL.md, **move** it (cut from SKILL.md, paste into the new sibling file, then leave a one-line reference in SKILL.md).
3. **Preserve author voice.** Do not rewrite prose for style. Only restructure for spec compliance. If frontmatter is incomplete, propose values that match the existing prose rather than inventing new framing.
4. **Atomic phases.** Run the five phases in order. Do not skip ahead — Phase 2 (directory rename) depends on Phase 1 (frontmatter `name`) being correct, and Phase 3 (body extraction) assumes Phase 1+2 are stable.
5. **Verify after each phase.** Re-read the modified files after edits land. Do not trust that an Edit succeeded without confirming the on-disk state.

## Procedure

### Phase 0 — Locate and inventory

1. Resolve the target path. Confirm it exists and is a directory.
2. List its contents (`ls -la <path>`). Record:
   - Whether `SKILL.md` exists at the root.
   - Which optional dirs are present: `scripts/`, `references/`, `assets/`.
   - Any unexpected files at the root (these are allowed by spec but worth flagging).
3. Read `SKILL.md` in full.
4. Parse the YAML frontmatter into a working map. If frontmatter is malformed (e.g. missing `---` fences, invalid YAML), stop and report the parse error — do not attempt repair until the user confirms how to proceed.
5. Compute the parent directory's basename.

Emit a brief inventory report:

```
Inventory: <path>
- SKILL.md: present | MISSING
- scripts/: present (N files) | absent
- references/: present (N files) | absent
- assets/: present (N files) | absent
- Other root files: <list>
- Frontmatter parse: OK | FAILED (<reason>)
- Body line count: <N>
- Parent dir basename: <name>
```

### Phase 1 — Frontmatter audit and fix

Validate each frontmatter field against the spec (see appendix). For each issue, propose a concrete fix before applying.

**Required field: `name`**

- Present? If missing, propose `<parent-dir-basename>` as the value.
- 1 to 64 characters?
- Allowed characters: lowercase alphanumeric (`a-z`, `0-9`) and hyphens (`-`) only? Reject uppercase, underscores, dots, spaces.
- Does not start or end with a hyphen?
- No consecutive hyphens (`--`)?
- Matches the parent directory basename **exactly**? If mismatched, this is the canonical signal for Phase 2 (rename either the directory or the `name` field — ask the user which).
- Has the form `<tier>-<slug>` where `<tier>` is one of `org`, `team`, `app`, or `project`? If the existing name lacks a recognized tier prefix:
  - Read the spec compliance level the user wants. If they want strict spec compliance, propose adding the appropriate tier prefix (and triggering a directory rename in Phase 2).
  - If the user has explicitly opted out of the tier-namespace convention for this skill (e.g. a meta-tool that doesn't fit the tier hierarchy), record that as an acknowledged deviation and skip the rename.

**Required field: `description`**

- Present and non-empty?
- 1 to 1024 characters?
- Describes both **what** the skill does and **when** to trigger it?
- Contains semantic trigger keywords (verbs and domain nouns the user would actually type)?

If the description is missing trigger phrasing, propose an expanded version that preserves the original "what" sentence and appends a "Trigger when …" clause synthesized from the body content. Show the diff before applying.

**Required field: `tier`**

- Present?
- Exactly one of `org`, `team`, `app`, `project` (lowercase)?

If missing, infer from context:
- `org` — broad cross-team applicability, generic tooling, no project-specific assumptions.
- `team` — shared by a team, may reference team-wide systems.
- `app` — scoped to a specific app or service.
- `project` — narrowest scope, single project.

Propose a tier and explain the reasoning; let the user confirm before writing.

**Optional fields**

For each optional field that is present, validate its shape. If absent, do not invent one — only add an optional field when the user explicitly asks or when the existing prose makes the value obvious (e.g. body literally documents a license).

- `license` — short string (license name or filename reference).
- `compatibility` — max 500 characters; environment / dependency notes.
- `metadata` — string-to-string map; check keys are unique.
- `allowed-tools` — space-delimited list (experimental field; preserve as-is if present).
- `dependencies` — YAML list; each entry matches `<name>[@<version-range>]` (in-repo) or `<owner>/<repo>#<name>[@<version-range>]` (cross-repo).

**Apply frontmatter fixes** using Edit (one field at a time, surgical replacements). After applying, re-Read the file and confirm the frontmatter now parses and validates.

### Phase 2 — Directory naming

If Phase 1 surfaced a `name` ↔ directory mismatch, resolve it now.

1. Present both options to the user:
   - **Rename the directory** to match the frontmatter `name` (preferred when the frontmatter value is correct and meaningful).
   - **Update the frontmatter `name`** to match the directory (preferred when the directory name is what callers already use).
2. Wait for confirmation. Never rename without explicit approval — the directory path may be referenced by other tooling.
3. Apply the chosen fix:
   - Directory rename: use `mv` via Bash. Note that this changes the working path — update any in-progress references.
   - Frontmatter update: Edit the `name:` line.
4. Re-verify the match.

### Phase 3 — Body audit and modularization

1. Count body lines (everything after the closing `---` of the frontmatter).
2. If under 500 lines and no extraction signals are present, skip to Phase 4.
3. **Extraction signals to look for:**
   - Inline code blocks longer than ~30 lines → candidates for `scripts/`.
   - Large reference tables, full API specifications, schemas, or domain glossaries → candidates for `references/`.
   - Template documents, config-file boilerplate, lookup tables, or any static resource the agent would reproduce verbatim → candidates for `assets/`.
4. For each extraction candidate, propose:
   - The destination file path (relative to the skill root).
   - The exact line range to cut from SKILL.md.
   - The replacement reference line (e.g. `See [scripts/extract.py](scripts/extract.py) for the extractor.`).
5. Get user confirmation before any extraction larger than 50 lines.
6. Apply extractions one at a time:
   - Read the candidate lines.
   - Write them to the new file (create the parent dir if needed).
   - Edit SKILL.md to replace the cut block with the reference line.
   - Re-Read SKILL.md and re-count lines.

**File reference rules** (verify after extraction):

- All file references in SKILL.md MUST use relative paths from the skill root (e.g. `scripts/extract.py`, not `/Users/.../scripts/extract.py` and not `./scripts/extract.py` with a redundant `./`).
- Reference chains should be one level deep — SKILL.md → `references/api.md` is fine; SKILL.md → `references/index.md` → `references/details.md` is not. Flag deeper chains and propose flattening.
- No absolute host-specific paths anywhere in the skill folder.

### Phase 4 — Optional-directory hygiene

For each present optional directory, verify its contents meet the spec:

- **`scripts/`**: each script should be self-contained or clearly document its dependencies in a top-of-file comment. Each script should handle edge cases gracefully and return helpful error messages. Do not rewrite scripts for style — only flag obvious gaps (missing shebang, no error handling on critical paths, hard-coded absolute paths). Ask before applying script-level changes.
- **`references/`**: each file should be focused and small (rough cap: ~200 lines per reference file — split larger ones).
- **`assets/`**: static resources only; no code, no executable content.

If any of these directories is missing but the body extraction in Phase 3 produced content of the corresponding type, create the directory and move the content there.

### Phase 5 — Final validation and report

Run the full validation matrix and emit a structured report.

**Structural checks**
- [ ] Skill directory exists at the resolved path
- [ ] `SKILL.md` is present and readable at the root
- [ ] Optional directories (if present) contain at least one file each

**Frontmatter checks**
- [ ] `name` present, 1–64 chars, lowercase alphanumeric + hyphens, no leading/trailing/consecutive hyphens
- [ ] `name` matches parent directory basename
- [ ] `name` has `<tier>-<slug>` form (or deviation is explicitly acknowledged)
- [ ] `description` present, non-empty, ≤ 1024 chars, includes both "what" and "when" framing
- [ ] `tier` present, one of `org` / `team` / `app` / `project`
- [ ] Any optional fields present are well-formed

**Body checks**
- [ ] Body is ≤ 500 lines
- [ ] No absolute file paths anywhere
- [ ] All relative file references resolve to existing files in the skill folder
- [ ] Reference chain depth ≤ 1

Output format:

```
Refactor report: <skill-name>
Location: <path>

Structural:    PASS | WARN | FAIL
Frontmatter:   PASS | WARN | FAIL
Body:          PASS | WARN | FAIL
Optional dirs: PASS | WARN | FAIL (or N/A)

Changes applied:
  - <change 1>
  - <change 2>
  ...

Outstanding issues (if any):
  - <issue> — <recommended next step>
```

## Edge cases

- **Frontmatter parse failure**: do not attempt repair. Show the raw text and the parser error; ask the user how to proceed.
- **`SKILL.md` missing entirely**: stop. This is a scaffolding task, not a refactor — redirect the user to `/skill-create`.
- **Multiple skills under one folder** (e.g. nested `SKILL.md` files): refactor only the top-level one. Flag the nested ones as separate refactor candidates.
- **Symlinked skill directory**: resolve the symlink first; refactor the underlying directory, not the link.
- **Skill is under version control with uncommitted changes**: warn the user before applying directory renames or large extractions; recommend committing the current state first so the refactor is a single reviewable diff.
- **Skill `name` already correct but directory wrong-cased on macOS**: macOS is case-insensitive but case-preserving — a rename from `MySkill` to `my-skill` will work, but a rename that differs only in case requires a two-step `mv` via a tmp name.
- **Optional fields the user wants to add but doesn't know values for**: ask once with a concrete proposal; do not loop on optional fields if the user defers — leave them absent.

## Examples

**Example 1 — Frontmatter-only fix**

Target: a skill whose `SKILL.md` has `name: my_skill` (underscore, invalid) and no `tier`. Body is fine.

Actions:
1. Phase 0 inventory shows valid SKILL.md, parsed frontmatter, body 120 lines.
2. Phase 1 flags `name` (underscore) and missing `tier`. Proposes `name: app-my-skill` (slug-ified, with inferred `app` tier from the body's app-specific references).
3. Phase 2 flags directory rename from `my_skill/` to `app-my-skill/`. User confirms.
4. Phases 3–4 no-op.
5. Phase 5 reports PASS on all checks.

**Example 2 — Body extraction**

Target: a skill with `SKILL.md` 780 lines, including a 400-line inline JSON schema reference and a 60-line shell script in a fenced code block.

Actions:
1. Phase 0 inventory shows body 780 lines (exceeds 500).
2. Phase 1 frontmatter is clean.
3. Phase 3 proposes: extract the JSON schema → `references/schema.md` (replace with one-line reference); extract the shell script → `scripts/deploy.sh` (replace with one-line reference). User confirms.
4. Apply both extractions. New body line count: 320.
5. Phase 4 creates `references/` and `scripts/` with the new files.
6. Phase 5 reports PASS, body now 320 lines, two new sibling files, all references relative.

## Appendix — AGENT SPEC reference

Inline summary of the spec this skill enforces. Authoritative source is the user-provided spec text.

**Directory contents (§2.2)**
- At minimum: `SKILL.md` at the root.
- Optional subdirectories: `scripts/`, `references/`, `assets/`. Any additional files or directories are allowed.

**SKILL.md (§3)**
- YAML frontmatter followed by Markdown body.

**Frontmatter fields (§3.1)**

| Field | Required | Constraint |
| --- | --- | --- |
| `name` | Yes | 1–64 chars, lowercase alphanumeric + hyphens, no leading/trailing/consecutive hyphens, must match parent dir, form `<tier>-<slug>` |
| `description` | Yes | 1–1024 chars, non-empty, describes what + when, semantic keywords |
| `tier` | Yes | One of `org`, `team`, `app`, `project` |
| `license` | No | Short string |
| `compatibility` | No | ≤ 500 chars |
| `metadata` | No | string→string map, unique keys |
| `allowed-tools` | No | Space-delimited list (experimental) |
| `dependencies` | No | YAML list of `<name>[@<version-range>]` or `<owner>/<repo>#<name>[@<version-range>]` |

**Body (§3.2)**
- Markdown, no strict format.
- SHOULD be ≤ 500 lines.
- File references MUST be relative from the skill root.
- Reference chains kept one level deep.

**Optional directories (§4)**
- `scripts/` — executable code; self-contained or documented deps; graceful error handling.
- `references/` — supplemental docs loaded on demand; small and focused.
- `assets/` — static resources (templates, schemas, lookup tables).

**Progressive disclosure (§5)**
- Startup (~100 tokens): only `name` + `description` loaded.
- Activation (< 5000 tokens): full `SKILL.md` body loaded.
- Execution: secondary resources loaded on demand.

This is why the 500-line body cap and the modularization into `scripts/`, `references/`, `assets/` matter — they keep activation cheap and push detail to on-demand loading.
