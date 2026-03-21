---
name: add-obsidian
description: Add Obsidian vault integration to NanoClaw. Mounts ~/obsidian-vault into agent containers so they can create, search, and manage notes.
---

# Add Obsidian Vault Integration

This skill connects the Obsidian vault at `~/obsidian-vault/` to NanoClaw agent containers, giving agents the ability to manage notes via the `obsidian-vault` container skill.

## Phase 1: Pre-flight

1. Verify the vault exists:

```bash
ls ~/obsidian-vault/
```

Expected: `Inbox/`, `Notes/`, `Projects/`, `Journal/`, `Daily/`, `Resources/`, `Templates/`, `.obsidian/`

If the vault doesn't exist, tell the user to create it first or offer to create the standard structure.

2. Check if the container skill already exists:

```bash
ls container/skills/obsidian-vault/SKILL.md
```

If it exists, skip to Phase 3 (configure mount).

## Phase 2: Create Container Skill

The container skill should already exist at `container/skills/obsidian-vault/SKILL.md` from the initial setup. If missing, read the template from this skill's directory and create it.

Verify:

```bash
head -5 container/skills/obsidian-vault/SKILL.md
```

Should show the `obsidian-vault` skill frontmatter.

## Phase 3: Configure Mount

### 3a. Update mount allowlist

Read the current allowlist:

```bash
cat ~/.config/nanoclaw/mount-allowlist.json
```

If `~/obsidian-vault` is not in `allowedRoots`, add it:

```json
{
  "path": "~/obsidian-vault",
  "allowReadWrite": true,
  "description": "Obsidian personal knowledge base"
}
```

Use `node -e` to read, modify, and write the JSON safely — do NOT use sed for JSON editing.

### 3b. Update main group container config

The main group needs `additionalMounts` configured to mount the vault. Update it via the database:

```bash
node -e "
const db = require('better-sqlite3')('data/nanoclaw.db');
const row = db.prepare('SELECT container_config FROM registered_groups WHERE is_main = 1').get();
const config = row?.container_config ? JSON.parse(row.container_config) : {};
const mounts = config.additionalMounts || [];
const hasObsidian = mounts.some(m => m.hostPath?.includes('obsidian-vault'));
if (!hasObsidian) {
  mounts.push({ hostPath: '/home/claude/obsidian-vault', containerPath: 'obsidian-vault', readonly: false });
  config.additionalMounts = mounts;
  db.prepare('UPDATE registered_groups SET container_config = ? WHERE is_main = 1').run(JSON.stringify(config));
  console.log('Added obsidian-vault mount to main group');
} else {
  console.log('obsidian-vault mount already configured');
}
"
```

If the database has no `registered_groups` table yet (NanoClaw not started), inform the user that the mount will be configured on first startup by adding a startup hook, OR instruct them to re-run this skill after NanoClaw has been started once.

### 3c. Add to other groups (optional)

If the user wants other groups to access the vault (read-only by default due to `nonMainReadOnly: true`), update their `container_config` similarly.

## Phase 4: Rebuild and Restart

### Rebuild container (to include the new skill)

```bash
./container/build.sh
```

### Restart NanoClaw

```bash
# Linux
systemctl --user restart nanoclaw

# macOS
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

## Phase 5: Verify

### Test from main chat

Send a message to the main group:

> "Create a test note in my Obsidian vault"

The agent should:
1. See the vault at `/workspace/extra/obsidian-vault/`
2. Create a note in `Inbox/` with proper frontmatter
3. Confirm creation

### Verify on host

```bash
ls ~/obsidian-vault/Inbox/
```

The test note should appear.

### Check capabilities

Send `/capabilities` in the main chat. The `obsidian-vault` skill should be listed.

## Troubleshooting

### Agent says vault not found

The mount is not configured or not validated. Check:

```bash
cat ~/.config/nanoclaw/mount-allowlist.json
```

Ensure `~/obsidian-vault` is in `allowedRoots` with `allowReadWrite: true`.

Then check the group's container config in the DB:

```bash
node -e "
const db = require('better-sqlite3')('data/nanoclaw.db');
const row = db.prepare('SELECT container_config FROM registered_groups WHERE is_main = 1').get();
console.log(row?.container_config || 'No config');
"
```

### Mount rejected in logs

Check NanoClaw logs for `Additional mount REJECTED`:

```bash
grep -i "mount.*REJECTED" logs/nanoclaw.log
```

Common causes:
- Path not in allowlist
- Path matches a blocked pattern
- Path doesn't exist on host

### Notes not syncing to mobile

This skill only handles the server-side vault and NanoClaw integration. For mobile sync, set up Syncthing separately to share `~/obsidian-vault/`.
