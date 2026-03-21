---
name: obsidian-vault
description: Manage the user's Obsidian vault — create, search, read, update, and organize notes. Use when the user mentions notes, ideas, journal, daily notes, or knowledge management.
---

# /obsidian-vault — Personal Knowledge Base

Manage the Obsidian vault mounted at `/workspace/extra/obsidian-vault/`.

## Vault Location

The vault is mounted at `/workspace/extra/obsidian-vault/`. Always use this path for all operations.

If the path does not exist, tell the user: "Your Obsidian vault is not mounted. Ask your admin to add the `obsidian-vault` additional mount to this group's container config."

## Folder Structure

| Folder | Purpose |
|--------|---------|
| `Inbox/` | Default landing zone for new notes. Quick captures go here. |
| `Notes/` | General reference notes, evergreen notes |
| `Projects/` | Project-specific notes. Use subfolders: `Projects/{project-name}/` |
| `Journal/` | Freeform journal entries |
| `Daily/` | Daily notes. One per day, named `YYYY-MM-DD.md` |
| `Resources/` | Links, references, saved articles |
| `Templates/` | Note templates (do NOT edit or delete these) |

## Creating Notes

### File Naming

- **Regular notes:** `kebab-case-title.md` (no date prefix unless contextually useful)
- **Daily notes:** `YYYY-MM-DD.md` in `Daily/`
- **Journal entries:** `YYYY-MM-DD-topic.md` in `Journal/`

### Required YAML Frontmatter

Every note MUST start with YAML frontmatter:

```yaml
---
title: "Human-readable title"
date: "YYYY-MM-DD HH:mm"
type: note | daily | meeting | project | journal | idea | reference
tags:
  - tag1
  - tag2
source: nanoclaw
---
```

- `date`: Use current date/time when creating. Format: `YYYY-MM-DD HH:mm`
- `type`: Choose the most appropriate type from the list above
- `source`: Always set to `nanoclaw` for agent-created notes
- `tags`: Include 1-5 relevant tags. Use lowercase, hyphenated (e.g., `machine-learning`)

### Content Guidelines

- Use standard Markdown
- Use `## Headings` for sections (not `# H1` — the title is in frontmatter)
- Use `[[Note Name]]` wiki-links to reference other notes
- Use `[[Note Name#Heading]]` to link to specific sections
- Use `#tag` inline sparingly — prefer tags in frontmatter
- Keep notes atomic: one idea or topic per note

## Daily Notes

When the user asks for a daily note:

1. Check if `Daily/YYYY-MM-DD.md` exists for today
2. If yes, append to it under the appropriate section
3. If no, create it with this structure:

```markdown
---
title: "YYYY-MM-DD Daily Note"
date: "YYYY-MM-DD"
type: daily
tags:
  - daily
source: nanoclaw
---

## Tasks
- [ ]

## Notes


## Journal

```

## Searching Notes

Use these tools for searching:

```bash
# Find notes by filename
Glob: /workspace/extra/obsidian-vault/**/*.md

# Search note contents
Grep: pattern in /workspace/extra/obsidian-vault/

# Search by tag in frontmatter
Grep: "- tag-name" in /workspace/extra/obsidian-vault/ glob *.md

# Search by type
Grep: "type: meeting" in /workspace/extra/obsidian-vault/ glob *.md
```

When presenting search results, show:
- Note title (from frontmatter or filename)
- Folder location
- Relevant excerpt (2-3 lines around match)
- Date

## Updating Notes

- Use the Edit tool to modify existing notes
- Always update the `date` field in frontmatter when making significant changes
- Never overwrite user-written content — append or insert at appropriate locations
- If adding to an existing section, append at the end of that section

## Organizing Notes

When asked to organize:

1. Move notes from `Inbox/` to appropriate folders
2. Add missing frontmatter to notes that lack it
3. Add wiki-links between related notes
4. Suggest merging duplicate or near-duplicate notes
5. Flag notes with no tags for review

## Conflict Resolution

If you find `.sync-conflict-*` files (from Syncthing):
1. Read both the original and conflict version
2. Show the user the differences
3. Ask which version to keep, or merge them
4. Delete the conflict file after resolution

## Quick Commands

The user may use shorthand:

| Command | Action |
|---------|--------|
| "save/note: [content]" | Create a note in Inbox/ |
| "daily" or "today" | Open or create today's daily note |
| "search [query]" | Search all notes |
| "organize" | Run organization pass on Inbox/ |
| "journal: [content]" | Create journal entry |
