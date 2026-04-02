---
name: release-notes
description: "Write release notes for Ask-Y platform after a stage-to-master merge. Researches git history across all repos, identifies the previous version, collects all PRs and commits, reads checked-in docs, asks the user for highlights, then writes structured release notes and a Slack announcement. Triggers on: release notes, write release notes, version notes, changelog, new version, what changed."
---

# Ask-Y Release Notes Skill

Automates the full release notes workflow for the Ask-Y data analytics platform after a stage-to-master (or stage-to-main) merge.

## IMPORTANT NOTES

- Always use **Windows paths** (`C:\`) and **Windows commands** (bash shell with forward slashes for git commands is fine)
- **Never run build or compile commands** — the user has explicitly asked not to
- The user will provide highlights and feedback — **always ask before writing the final notes**
- Release notes follow a specific format — match the style of the most recent existing release notes exactly

## REPOSITORY LOCATIONS

There are **three repositories** that make up a release:

| Repo | Path | Branch Model | Purpose |
|------|------|-------------|---------|
| **jamback** (Backend) | `C:\work\asky\jam\jamback\` | `stage` → `master` | .NET backend, APIs, services, tools |
| **prismfront** (Frontend) | `C:\work\asky\prism\prismfront\` | `stage` → `master` | Angular frontend UI |
| **asky-lib** (RiffML) | `C:\work\asky-workspaces\asky-lib\` | `stage` → `main` | LLM agent instructions (riff files) |

Note: asky-lib uses `main` not `master`.

## STEP 1: Identify the Previous Version

Read the most recent release notes file in jamback to find the previous version number and its date:

```bash
ls -t /c/work/asky/jam/jamback/RELEASE_NOTES_v*.md | head -5
```

Read the latest one to get:
- The **version number** (e.g., v2.3 → next is v2.4)
- The **release date** (this is the start date for the new changelog)
- The **format and structure** to follow

## STEP 2: Find the Last Merge Date

Find when stage was last merged to master/main before the current merge in each repo:

**jamback:**
```bash
cd /c/work/asky/jam/jamback && git log master --oneline --date=short --pretty=format:"%h %ad %s" -50
```

**prismfront:**
```bash
cd /c/work/asky/prism/prismfront && git log master --oneline --date=short --pretty=format:"%h %ad %s" -50
```

**asky-lib:**
```bash
cd /c/work/asky-workspaces/asky-lib && git log main --oneline --date=short --pretty=format:"%h %ad %s" -50
```

Look for the merge commit just before the latest "Merge branch 'stage'" — the date of the second-most-recent merge is where the new changelog starts.

## STEP 3: Collect All Changes

For each repo, get the full commit log since the last merge date. Run all three in parallel:

**All commits (for detail):**
```bash
cd /c/work/asky/jam/jamback && git log stage --since="YYYY-MM-DD" --date=short --pretty=format:"%h %ad %s"
```

**PR merges only (for structure):**
```bash
cd /c/work/asky/jam/jamback && git log stage --since="YYYY-MM-DD" --merges --oneline --date=short --pretty=format:"%h %ad %s"
```

Repeat for prismfront and asky-lib with their respective paths.

**Count totals for the summary:**
```bash
# Per repo: commit count and PR count
git log stage --since="YYYY-MM-DD" --oneline | wc -l
git log stage --since="YYYY-MM-DD" --merges --grep="Merge pull request" --oneline | wc -l
```

## STEP 4: Read Checked-In Documentation

Find and read any new or updated docs since the last release:

```bash
cd /c/work/asky/jam/jamback && find . -name "*.md" -newer RELEASE_NOTES_v{PREV}.md -not -path "./.git/*"
```

**Read every doc found.** These contain detailed architectural descriptions of new features and are critical for writing accurate release notes. Key docs are typically in `./docs/`.

Also check the RiffML files for agent instruction changes:

```bash
ls /c/work/asky-workspaces/asky-lib/riffml/prism/
ls /c/work/asky-workspaces/asky-lib/riffml/prism/Agents/
```

If a new agent version directory exists (e.g., `prism5/` when previous was `prism4/`), note this as a major change.

## STEP 5: Write the Full Changelog File

Before writing release notes, dump the raw commit log to a file for reference:

Create `v{VERSION}_full_changelog.md` in the jamback repo root with all commits from all three repos, organized by repo:

```
=== JAMBACK (Backend) - Commits since YYYY-MM-DD ===
- hash date | commit message
...

=== PRISMFRONT (Frontend) - Commits since YYYY-MM-DD ===
...

=== ASKY-LIB (RiffML Agent Instructions) - Commits since YYYY-MM-DD ===
...
```

## STEP 6: Ask the User for Highlights and Feedback

**This step is mandatory — do not skip it.**

Before writing the release notes, present the user with:

1. A summary of the major feature areas you identified from the git research
2. The proposed version number
3. Ask them:
   - What are the **key highlights** they want to emphasize?
   - Any features that should be **called out or downplayed**?
   - Any **team members to congratulate** in the Slack message?
   - Any features that are **infrastructure only** (not user-facing yet)?
   - Any **corrections** to your understanding of the changes?

Wait for their response before proceeding to Step 7.

## STEP 7: Write the Release Notes

Create `RELEASE_NOTES_v{VERSION}.md` in the jamback repo root following this exact structure:

```markdown
# Ask-Y Prism - Release Notes - Version {X.Y}
**Release Date:** {Month Day, Year}
**Period Covered:** {Start Date} - {End Date}

---

## New Features

### {Feature Name}
{2-3 sentence description of what it is and why it matters}

- **{Sub-feature}**: {Detail}
- **{Sub-feature}**: {Detail}
...

---

## Improvements

### {Category}
- **{Item}**: {Detail}
...

---

## Bug Fixes

- Fixed {description}
...

---

## Technical Updates

### Dependencies & Infrastructure
- **{Item}**: {Detail}

### Architecture Changes
- {Description of architectural change}

### RiffML Agent Instructions
- **{Item}**: {Detail}

**Total Changes**: ~{N} commits, {M} pull requests merged across backend, frontend, and agent instructions

For the detailed commit-by-commit changelog, see `v{VERSION}_full_changelog.md`.
```

### Writing Guidelines

- **New Features**: Major user-facing capabilities. Each gets its own H3 section with a description paragraph and bullet points for sub-features. Use bold for sub-feature names.
- **Improvements**: Enhancements to existing features, grouped by category. Less prominent than new features.
- **Bug Fixes**: Simple bullet list, each starting with "Fixed". No need to explain the root cause.
- **Technical Updates**: Internal changes relevant to developers. Group by Dependencies, Architecture, and RiffML.
- **Tone**: Professional but approachable. Describe what the feature does for the user, not just what was coded.
- **Naming**: Use the product terminology the user confirms (e.g., "Skills" not "Knowledge Books", "Prisms" not "Views").
- **Infrastructure features**: If the user says something is infra-only, mention it but note it's laying groundwork for the next version.

## STEP 8: Write the Slack Announcement

After the release notes are approved, write a short, friendly Slack message:

### Slack Message Guidelines

- Start with a greeting and the version announcement with a celebration emoji
- **Bold** feature names as section headers
- Keep descriptions to 1-2 sentences each — much shorter than the release notes
- Include any **team shoutouts** the user requested
- For infra-only features, phrase as "coming in v{NEXT}" or "ready for the next version"
- End with total stats (commits, PRs) and a rocket emoji
- Tone: friendly, celebratory, concise — this is a team chat, not a formal document
- Do NOT use bullet points for every feature — use short paragraphs with bold headers

### Slack Message Template

```
Hey team! 🎉

**Ask-Y v{X.Y} is live!** Just merged stage to master across all repos. Here's what's new:

**{Feature 1}** — {1-2 sentence description}

**{Feature 2}** — {1-2 sentence description}

{Shoutout to team member if requested}

**{Feature N}** — {description}

Plus: {quick list of smaller improvements}.

~{N} commits, {M} PRs. Nice work everyone! 🚀
```

## COMMON PATTERNS TO LOOK FOR IN GIT LOGS

When analyzing commits, look for these patterns to identify features:

| Pattern in Commits | Likely Feature |
|---|---|
| "knowledge book" → "skill" renames | Rebranding/redesign |
| "widget", "iteration", "DAG" | Agent visualization |
| "web search", "tavily", "brave" | Web search integration |
| "prompts library", "resolve-references" | Prompt management |
| "connector", "connected tables" | Database connectors |
| "prism5", "prism6", etc. | Agent framework upgrade |
| "replay", "flow view" | Debugger improvements |
| "e2e", "data-qa", "test" | Testing infrastructure |
| "semantic", "unification" | Data modeling features |
| "streaming", "progress" | UX improvements |
| "report", "dynamic" | Reporting features |

## VERSION NUMBERING

- Major features (new agent framework, new UI paradigm) → bump minor version (2.3 → 2.4)
- The user will confirm the version number, but default to incrementing the minor version by 0.1
