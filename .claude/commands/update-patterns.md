---
description: Update prompt patterns from official documentation and source repositories
argument-hint: "[--analyze-only] [--skip-repo-update] [--skip-docs-fetch]"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Grep", "WebFetch", "Bash(git clone:*)", "Bash(git -C:*)", "Bash(git describe:*)", "Bash(git rev-parse:*)", "Bash(git pull:*)", "Bash(date:*)", "Bash(awk:*)", "Bash(jq:*)", "Bash(rm -rf:*)", "Task", "TodoWrite"]
---

# Update Patterns Command

Analyze the official Claude Code documentation and source repositories, then update prompt-smith patterns.

**Important**: This is a development workspace command. It updates the plugin source in the linked my-claude-plugins repository.

## Priority Rule

**When official documentation and repository analysis contradict each other, official documentation always takes precedence.**

- Official documentation = the specification (what is correct)
- Repository analysis = observed patterns (what is common)

Examples of contradictions:
- Documentation says a field is optional, but all repo examples include it → treat as optional, note that it is commonly used
- Documentation lists a field not seen in any repo example → include it as a valid field
- Repo examples use a field not mentioned in documentation → flag as undocumented, do not recommend

## Prerequisites

This command expects the my-claude-plugins repository to be accessible. Determine the plugin root path:

1. Check if a symlink or directory exists at `../my-claude-plugins/plugins/prompt-smith/`
2. If not found, ask the user for the path to the my-claude-plugins repository

Store the resolved path as `{PLUGIN_ROOT}` for use throughout this command.

## Sources

| Source | Type | Description |
|--------|------|-------------|
| Claude Code official docs | Specification | Authoritative reference for fields, behavior, and best practices |
| `claude-code` repo | Repository | Plugins bundled with Claude Code |
| `claude-plugins-official` repo | Repository | Anthropic official plugin directory |

## Input

**Options:** $ARGUMENTS

- `--analyze-only`: Run analysis only, no file updates
- `--skip-docs-fetch`: Skip official documentation fetch (Phase 1), use existing `docs/official-reference.md`
- `--skip-repo-update`: Skip repository clone/pull (Phase 2)

**If $ARGUMENTS is empty:** Run full update (all phases, no skips).

## Process

### Phase 1: Official Documentation Analysis

**Goal**: Extract the latest specification from official Claude Code documentation (authoritative source)

**Skip if** `--skip-docs-fetch` is specified. In that case, read the existing `docs/official-reference.md` instead.

**Actions**:

Fetch the following pages using WebFetch and extract all specification details:

| URL | Target |
|-----|--------|
| `https://docs.anthropic.com/en/docs/claude-code/skills` | Skill specification |
| `https://docs.anthropic.com/en/docs/claude-code/sub-agents` | Agent specification |
| `https://docs.anthropic.com/en/docs/claude-code/plugins` | Plugin overview |
| `https://docs.anthropic.com/en/docs/claude-code/plugins-reference` | Plugin reference |

For each page, extract:
- All YAML frontmatter fields, their types, required/optional status, and default values
- Valid values and constraints for each field
- File structure requirements
- Best practices and recommendations
- Behavioral rules (e.g., invocation control, context loading)
- Any new features or deprecated fields

**Output format** (internal, used in later phases):
```markdown
## Official Specification: {Category}

### Frontmatter Fields
| Field | Required | Default | Type | Description |
|-------|----------|---------|------|-------------|
| {field} | {Yes/No/Recommended} | {default} | {type} | {description} |

### File Structure
- {requirement}

### Behavioral Rules
- {rule}

### Best Practices
- {practice}

### New/Changed Since Last Update
- {change}
```

### Phase 2: Repository Update & Pattern Analysis

**Goal**: Get the latest source repositories and analyze observed patterns

**Skip if** `--skip-repo-update` is specified.

**Actions**:

#### 2a: Clone/Update Repositories

1. Create `.refs/` directory if it doesn't exist
2. Clone or update repositories:
   - If `.refs/claude-code` doesn't exist:
     `git clone --depth 1 https://github.com/anthropics/claude-code.git .refs/claude-code`
   - Else:
     `git -C .refs/claude-code pull`
   - If `.refs/claude-plugins-official` doesn't exist:
     `git clone --depth 1 https://github.com/anthropics/claude-plugins-official.git .refs/claude-plugins-official`
   - Else:
     `git -C .refs/claude-plugins-official pull`
3. Show latest commits for each:
   - `git -C .refs/claude-code log --oneline -5`
   - `git -C .refs/claude-plugins-official log --oneline -5`

#### 2b: Pattern Analysis

Launch 3 agents in parallel to analyze (each agent searches both repositories).

Each agent must return results in the following format:
```markdown
## {Category} Analysis

### Files Analyzed
- {file path} (from {repo})

### Frontmatter Fields
| Field | Frequency | Example |
|-------|-----------|---------|
| {field} | {n}/{total} ({%}) | {value} |

### Structural Patterns
- {pattern name}: {description} ({frequency})

### Notable Conventions
- {convention}
```

1. **Skills Analysis Agent** (haiku):
   - Target:
     - `.refs/claude-code/plugins/*/skills/*/SKILL.md`
     - `.refs/claude-plugins-official/plugins/*/skills/*/SKILL.md`
   - Extract: YAML frontmatter patterns, section structure, description format

2. **Agents Analysis Agent** (haiku):
   - Target:
     - `.refs/claude-code/plugins/*/agents/*.md`
     - `.refs/claude-plugins-official/plugins/*/agents/*.md`
   - Extract: YAML frontmatter patterns, example block structure, model/tools patterns

3. **Commands Analysis Agent** (haiku):
   - Target:
     - `.refs/claude-code/plugins/*/commands/*.md` and `.refs/claude-code/.claude/commands/*.md`
     - `.refs/claude-plugins-official/plugins/*/commands/*.md`
   - **Exclude**: `.refs/claude-plugins-official/external_plugins/` (third-party)
   - Extract: YAML frontmatter patterns, workflow patterns, `$ARGUMENTS` handling

### Phase 3: Cross-Reference and Compare

**Goal**: Identify differences, contradictions, and updates needed

**Actions**:

1. Read current reference files:
   - `docs/official-reference.md` (current official spec snapshot)
   - `{PLUGIN_ROOT}/skills/prompt-creation/references/patterns.md` (current patterns)

2. **Compare official docs (Phase 1) against current `docs/official-reference.md`**:
   - New fields added in official docs
   - Fields removed or deprecated
   - Changed descriptions, defaults, or constraints
   - New best practices or behavioral rules

3. **Compare repo analysis (Phase 2b) against current patterns**:
   - New patterns discovered
   - Modified patterns (frequency changes, new conventions)
   - Deprecated patterns (no longer observed)

4. **Cross-reference official docs vs repo analysis** — flag contradictions:
   - Fields documented but never used in repos → valid but uncommon
   - Fields used in repos but not in documentation → undocumented, flag for review
   - Behavioral differences between spec and practice → note spec as authoritative

**Output**: Structured list of changes categorized as:
- `[SPEC]` — change from official documentation (authoritative)
- `[PATTERN]` — change from repository analysis (observational)
- `[CONFLICT]` — contradiction between spec and pattern (spec wins, note the discrepancy)

### Phase 4: Report

**Always show this report.** If `--analyze-only`, stop after showing the report.

**Output format**:
```markdown
# Pattern Analysis Report

## Summary
- Official doc pages fetched: {count}
- Skills analyzed: {count} (from repos)
- Agents analyzed: {count} (from repos)
- Commands analyzed: {count} (from repos)

## Specification Changes (from official docs)

### New
- [SPEC] {description}

### Modified
- [SPEC] {what changed}

### Removed/Deprecated
- [SPEC] {what was removed}

## Pattern Changes (from repo analysis)

### New
- [PATTERN] {description}

### Modified
- [PATTERN] {what changed}

### No Changes
- {unchanged areas}

## Conflicts (official docs vs repo patterns)
- [CONFLICT] {field/behavior}: Official says {X}, repos show {Y} → **Using official spec**

## Recommended Updates
1. {specific update recommendation with source tag}
2. {specific update recommendation with source tag}
```

### Phase 5: Update Files

**Goal**: Update prompt-smith reference docs and plugin pattern definitions

**Skip if** `--analyze-only` is specified.

**Actions**:

1. **Update `docs/official-reference.md`** (this workspace):
   - Path: `docs/official-reference.md`
   - Update with latest specification from Phase 1
   - Update the `最終更新` date at the top of the file

2. **Update SKILL.md**:
   - Path: `{PLUGIN_ROOT}/skills/prompt-creation/SKILL.md`
   - Update Quick Reference section if patterns changed
   - Ensure field information aligns with official spec

3. **Update patterns.md**:
   - Path: `{PLUGIN_ROOT}/skills/prompt-creation/references/patterns.md`
   - Update detailed patterns and checklists
   - For any [CONFLICT] items: use official spec, add note about repo divergence

4. **Update review-prompt.md** (if criteria changed):
   - Path: `{PLUGIN_ROOT}/commands/review-prompt.md`
   - Update review criteria sections
   - Review criteria must reflect official spec, not just observed patterns

5. **Show diff summary**:
   - List files modified (both workspace and plugin)
   - Show key changes

### Phase 6: Verification

**Goal**: Ensure updates are valid

**Tools**: Use `Grep`/`Read` tools for content checks, `awk`/`jq` via Bash for structured validation. Do NOT install additional dependencies (e.g., `pip install`).

**Actions**:

1. **Validate YAML frontmatter** in each updated file:
   - Use `Grep` to check for required fields (`^name:`, `^description:`) within the first 10 lines
   - SKILL.md: `name`, `description` required
   - Agent .md: `name`, `description` required
   - Command .md: `description` required
   - patterns.md: no frontmatter required (skip)

2. **Check required sections** exist using `Grep`:
   - patterns.md: `^## Skill Patterns`, `^## Agent Patterns`, `^## Command Patterns`, `^## Plugin Patterns`, `^## Checklists`

3. **Validate JSON** in plugin.json using `jq`:

   ```bash
   jq . {PLUGIN_ROOT}/.claude-plugin/plugin.json > /dev/null
   ```

4. **Verify official-reference.md** was updated:
   - Check `最終更新` date matches today
   - Check all 4 main sections exist: `^## 1. Skill`, `^## 2. Agent`, `^## 3. Command`, `^## 4. Plugin`

**Output**:
```markdown
# Update Complete

## Files Updated
- docs/official-reference.md: {summary of changes}
- {PLUGIN_ROOT}/...: {summary of changes}

## Verification
- YAML validity: OK
- Required sections: OK
- JSON validity: OK
- Official reference updated: OK

## Next Steps
- Review the changes manually if needed
- Test with `/prompt-smith:review-prompt` on sample files
```

### Phase 7: Version Update

**Goal**: Update version, source tracking info, and changelog

**Skip if** `--analyze-only` is specified.

**Actions**:
1. Get today's date: `date +%Y-%m-%d`
2. Get current HEAD info for each repository:
   - claude-code:
     - Commit hash: `git -C .refs/claude-code rev-parse --short HEAD`
     - Tag (if available): `git -C .refs/claude-code describe --tags --always`
   - claude-plugins-official:
     - Commit hash: `git -C .refs/claude-plugins-official rev-parse --short HEAD`
     - Tag (if available): `git -C .refs/claude-plugins-official describe --tags --always`
   - Tag resolution rules:
     - If the commit has an exact tag: use the tag (e.g., `v1.0.20`)
     - If no exact tag: use the describe output (e.g., `v1.0.20-15-gdfd3494`)
     - If no tags at all: omit tag field
3. Read `{PLUGIN_ROOT}/.claude-plugin/plugin.json`
4. Bump patch version (e.g., 2.0.0 → 2.0.1)
5. Write updated `plugin.json`
6. Generate `{PLUGIN_ROOT}/.claude-plugin/version-info.txt` from plugin.json and Step 2 results:
   ```
   prompt-smith v{version}
   {description}

   Author: {author.name} <{author.email}>
   Sources:
     - docs.anthropic.com/en/docs/claude-code/ (fetched: {date})
     - anthropics/claude-code@{commit}{ " ({tag})" if tag exists}
     - anthropics/claude-plugins-official@{commit}{ " ({tag})" if tag exists}
   Analyzed at: {date from Step 1}
   ```
   - `tag` が存在する場合のみ括弧付きで表示、なければコミットハッシュのみ
7. Prepend new entry to `{PLUGIN_ROOT}/CHANGELOG.md` (write in Japanese):
   ```markdown
   ## [{new_version}] - {date}
   - 公式ドキュメント仕様を反映
   - パターン定義を更新 (claude-code@{commit}, claude-plugins-official@{commit})
   - {list of specific changes from Phase 3, in Japanese, with [SPEC]/[PATTERN]/[CONFLICT] tags}
   ```

## Error Handling

**If official documentation fetch fails:**
1. Report which URLs failed
2. Fall back to existing `docs/official-reference.md` for that category
3. Add warning to report that spec data may be stale

**If repository clone/pull fails:**
1. Check network connectivity
2. Suggest manual clone:
   - `git clone --depth 1 https://github.com/anthropics/claude-code.git .refs/claude-code`
   - `git clone --depth 1 https://github.com/anthropics/claude-plugins-official.git .refs/claude-plugins-official`

**If my-claude-plugins not found:**
- Ask the user for the path

**If no changes detected:**
- Report "Patterns are up to date"
- Show last analysis date if available

**If analysis fails:**
- Report which category/source failed
- Continue with successful analyses
- Show partial results

## Example Usage

```bash
# Full update (docs + repos)
/update-patterns

# Analysis only (no file changes)
/update-patterns --analyze-only

# Skip git operations (use existing repository state)
/update-patterns --skip-repo-update

# Skip docs fetch (use existing official-reference.md)
/update-patterns --skip-docs-fetch

# Skip both fetches (pure comparison with existing data)
/update-patterns --skip-repo-update --skip-docs-fetch
```
