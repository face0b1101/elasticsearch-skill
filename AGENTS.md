# Project Rules

> **Single Source of Truth** for AI coding assistants (Claude Code, Cursor, etc.)

______________________________________________________________________

## AT STARTUP

1. Check if `./.beads` folder exists.
   If it exists, this project uses **beads (bd)** for issue tracking.

   - Run `bd ready` to see available work before starting
   - Reference issues in commits: `[bd-XX] description`

2. Check if `./specify/` folder exists.
   If it exists, this project uses **speckit** for specification-driven development.

   - See the **Project Planning & Tracking** section below for speckit workflow details.

______________________________________________________________________

## Issue Tracking (if `.beads/` exists)

This project can use `bd` (beads) for lightweight issue tracking with dependency support.

**Check if enabled**: Look for `.beads/` folder in project root.

**If enabled**, use these commands:

```bash
bd ready                    # What can I work on? (no blockers)
bd list --status open       # All open issues
bd create "title" --type bug  # Create new issue
bd close [id] -r "reason"   # Close with reason
```

**Workflow integration**:

- Before starting work: `bd ready`
- Reference in commits: `[bd-XX] description`
- After completing: `bd close bd-XX -r "Done"`

______________________________________________________________________

## Releasing to GitHub

This project ships as a **zip of user-facing files** attached to a GitHub Release. The zip is built ephemerally — it is never committed to the repo.

### Files included in the release zip

Only these files belong in the zip:

| File           | Purpose                       |
| -------------- | ----------------------------- |
| `SKILL.md`     | The skill (core deliverable)  |
| `references/`  | Deep-dive API reference files |
| `README.md`    | Installation and usage guide  |
| `LICENCE`      | CC BY 4.0 licence text        |
| `CHANGELOG.md` | Release history               |

Do **not** include `AGENTS.md`, dotfiles (`.gitignore`, `.markdownlint.json`), or any other development artefact.

### Release workflow

1. **Update `CHANGELOG.md`** — move items from `[Unreleased]` into a new version heading with today's date. Follow [Keep a Changelog](https://keepachangelog.com/) format.

2. **Commit** the changelog (and any other release-ready changes) with message `release: vX.Y.Z — <summary>`.

3. **Tag** with an annotated tag: `git tag -a vX.Y.Z -m "vX.Y.Z"`.

4. **Push** commit and tag: `git push origin main --tags`.

5. **Build the zip** in `/tmp` (not in the repo):

   ```bash
   zip -r /tmp/elasticsearch-skill-vX.Y.Z.zip SKILL.md references/ README.md LICENCE CHANGELOG.md
   ```

6. **Create the GitHub Release** with the zip attached:

   ```bash
   gh release create vX.Y.Z /tmp/elasticsearch-skill-vX.Y.Z.zip \
     --title "vX.Y.Z" \
     --notes "<release notes from CHANGELOG>"
   ```

### Important notes

- The `releases/latest` URL in `README.md` resolves automatically — no need to update it.
- GitHub adds "Source code" archives to every release by default; these cannot be removed. The uploaded zip is the intended download for end users.
- Version numbers track the Elastic Stack version the skill is tested against (e.g. `v9.3.1` means tested against Elastic Stack 9.3.1). Skill-only fixes for the same stack version append a suffix (e.g. `v9.3.1-1`). Prior releases used independent semantic versioning (v1.0.0, v1.1.0).

______________________________________________________________________

## Reviewing Against a New Stack Version

When a new Elastic Stack version is released, follow this process to check and update the skill.

### 1. Gather breaking changes

- Fetch the Elastic breaking changes page for the target version range: `elastic.co/docs/release-notes/elasticsearch/breaking-changes`
- Search for breaking changes affecting: REST API endpoints, Query DSL, aggregations, ingest processors, mapping definitions, cluster APIs, ES|QL syntax, and Kibana APIs
- Note which changes are **removals** (will cause errors) vs **behavioural** (may cause unexpected results)

### 2. Audit skill content against breaking changes

- Read `SKILL.md` and every file in `references/`
- For each breaking change, search the skill files for affected API endpoints, parameter names, or feature names
- Categorise findings:
  - **Critical** — code examples that would fail against the new version
  - **Important** — accuracy or completeness issues
  - **No change needed** — skill already uses the current syntax

### 3. Research new features

- Check release notes and blogs for each minor version in the range (e.g. 9.0, 9.1, 9.2, 9.3)
- Focus on: new ES|QL commands reaching GA, new REST APIs, features moving from preview to GA, new Kibana APIs
- Decide which are significant enough to add to the skill (GA features that users would commonly need)

### 4. Apply updates

- Fix critical issues first (broken examples)
- Update version references and accuracy issues
- Add new feature coverage where warranted
- Update the `Tested against` line in `SKILL.md`

### 5. Release

- Follow the release workflow above with the new stack-aligned version number

______________________________________________________________________

## Project Planning & Tracking (if `.specify/` exists)

This project can use [speckit](https://speckit.org) for AI-powered specification-driven development.

**Check if enabled**: Look for `.specify/` folder in project root.

**If enabled**, encourage the user to use this workflow for new features:

1. Use Speckit to define spec → plan → tasks (`/speckit.constitution`, `/speckit.specify`, `/speckit.plan`, `/speckit.tasks`)
2. Track tasks:
   - **If Beads is enabled** (`.beads/` exists): Convert tasks to Beads issues (`bd create`)
   - **Otherwise**: Track tasks manually or use GitHub Issues
3. **If using Beads**: Add dependencies between tasks (`bd dep add`)
4. Implement using `/speckit.implement`
5. Update task status as you progress:
   - **With Beads**: `bd update`, `bd close`
   - **Without Beads**: Update your tracking system manually
6. **If using Beads**: File new issues for discovered work (`bd create --type discovered-from`)

**Check ready work**:

- **With Beads**: `bd ready --json`
- **Without Beads**: Review your task list manually

______________________________________________________________________

## AI Assistant Operating Rules

Concise policy reference for all coding agents touching this repository. Keep responses factual and avoid speculative language.

### 1. Communication & Planning

- Always mention assumptions; ask the user to confirm anything ambiguous before editing.
- Follow the required plan/approval workflow when prompted and wait for explicit approval to execute.
- Use UK-English spelling in comments, documentation, and commit messages.

### 2. File Safety

- Do **not** edit `.env` or other environment files; only reference `.env.example`.
- Delete files only when you created them or the user explicitly instructs you to remove older assets.
- Never run destructive git commands (`git reset --hard`, `git checkout --`, `git restore`, `rm -rf .git`) unless the user provides written approval in this thread.
- Treat rename automation as a one-time setup; never re-run it on an established project.

### 3. Collaboration Etiquette

- If another agent has edited a file, read their changes and build on them-do not revert or overwrite.
- Coordinate before touching large refactors that might conflict with ongoing work.
- Keep diffs minimal and reviewable; use targeted edits rather than rewriting whole files.

### 4. Git & Commits

- Check `git status` before staging and before committing.
- Keep commits atomic and list paths explicitly, e.g. `git commit -m "feat: add CI" -- path/to/file`.
- For new files: `git restore --staged :/ && git add <paths> && git commit -m "<msg>" -- <paths>`.
- Quote any paths containing brackets/parentheses when staging to avoid globbing.
- Never amend existing commits unless the user instructs you to.
- Don't plaster all commits and git issues with "Made with Cursor", "Cursor helped me with this", "AI did everything" or anything similar.

### 5. Pre-flight Checklist

1. Read the task, confirm assumptions, and outline the approach.
2. Inspect the relevant files (include imports/configs for context).
3. Review changes for spelling, formatting, and consistency before committing.
4. Summarise edits, mention tests, and flag follow-up work in the final response.
