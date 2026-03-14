---
name: add-google-workspace
description: Add Google Workspace integration to NanoClaw using the official gws CLI. Provides access to Gmail, Drive, Calendar, Docs, Sheets, Chat, and more via MCP. Guides through OAuth setup and service selection.
---

# Add Google Workspace Integration

This skill adds Google Workspace support to NanoClaw using the official Google Workspace CLI (`gws`). It exposes Gmail, Drive, Calendar, Docs, Sheets, Chat, and more as MCP tools available to all agents.

**Why gws over add-gmail?**
- Official Google tool (not third-party)
- Supports all Google Workspace services, not just Gmail
- Native MCP support
- Enterprise-grade security (AES-256-GCM encrypted credentials)

> **Conflicts with add-gmail:** If `add-gmail` is already installed, this skill provides a superset — you may remove Gmail's separate MCP server after setup.

## Phase 1: Pre-flight

### Check if already applied

Check `container/agent-runner/src/index.ts` for `mcp__gws__*`. If found, skip to Phase 3 (Setup).

### Ask the user which services to enable

Use `AskUserQuestion`:

> Which Google Workspace services should agents have access to?
> 1. Gmail only
> 2. Gmail + Drive
> 3. Gmail + Drive + Calendar
> 4. Full Workspace (Gmail, Drive, Calendar, Docs, Sheets, Chat, Meet)

Record the chosen services as a comma-separated list for the `-s` flag:
- Option 1: `gmail`
- Option 2: `gmail,drive`
- Option 3: `gmail,drive,calendar`
- Option 4: `gmail,drive,calendar,docs,sheets,chat,meet`

## Phase 2: Apply Code Changes

### 1. Mount gws config directory in `src/container-runner.ts`

Add `import os from 'os';` to the imports if not already present (after the existing `import fs from 'fs';` line).

Then in `buildVolumeMounts`, after the `.claude` session mount (the block ending with `containerPath: '/home/node/.claude'`) and before the IPC mount section, add:

```typescript
  // Google Workspace CLI credentials (shared across all groups)
  const gwsConfigDir = path.join(os.homedir(), '.config', 'gws');
  if (fs.existsSync(gwsConfigDir)) {
    mounts.push({
      hostPath: gwsConfigDir,
      containerPath: '/home/node/.config/gws',
      readonly: false,
    });
  }
```

### 2. Register gws MCP server in `container/agent-runner/src/index.ts`

In the `allowedTools` array, add `'mcp__gws__*'` after `'mcp__nanoclaw__*'`:

```typescript
allowedTools: [
  'Bash',
  'Read', 'Write', 'Edit', 'Glob', 'Grep',
  'WebSearch', 'WebFetch',
  'Task', 'TaskOutput', 'TaskStop',
  'TeamCreate', 'TeamDelete', 'SendMessage',
  'TodoWrite', 'ToolSearch', 'Skill',
  'NotebookEdit',
  'mcp__nanoclaw__*',
  'mcp__gws__*'
],
```

In the `mcpServers` object, add the `gws` server after the `nanoclaw` entry (replace `SERVICES` with the user's chosen services list):

```typescript
mcpServers: {
  nanoclaw: {
    command: 'node',
    args: [mcpServerPath],
    env: {
      NANOCLAW_CHAT_JID: containerInput.chatJid,
      NANOCLAW_GROUP_FOLDER: containerInput.groupFolder,
      NANOCLAW_IS_MAIN: containerInput.isMain ? '1' : '0',
    },
  },
  gws: {
    command: 'gws',
    args: ['mcp', '-s', 'SERVICES'],
  },
},
```

### 3. Validate build

```bash
npm run build
```

Build must be clean before proceeding.

## Phase 3: Setup

### Install gws CLI

```bash
npm install -g @googleworkspace/cli
gws --version
```

If the install fails, try: `sudo npm install -g @googleworkspace/cli`

### Check for existing credentials

```bash
ls -la ~/.config/gws/ 2>/dev/null || echo "No gws config found"
```

If credentials exist, skip to "Build and restart".

### GCP OAuth Setup

Tell the user:

> To connect Google Workspace, I need OAuth credentials from Google Cloud:
>
> 1. Open https://console.cloud.google.com — create or select a project
> 2. Go to **APIs & Services > Library** and enable the APIs for your chosen services:
>    - Gmail API (for gmail)
>    - Google Drive API (for drive)
>    - Google Calendar API (for calendar)
>    - Additional APIs as needed
> 3. Go to **APIs & Services > Credentials > + CREATE CREDENTIALS > OAuth client ID**
>    - If prompted for consent screen: choose "External", fill in app name and email, save
>    - Application type: **Desktop app**, name: e.g. "NanoClaw GWS"
> 4. Download the JSON credentials file
>
> What is the path to the downloaded credentials file?

Copy the credentials:

```bash
mkdir -p ~/.config/gws
cp "/path/user/provided" ~/.config/gws/credentials.json
```

### Authorize via browser

Tell the user:

> I'll run the Google Workspace authorization now. A browser window will open — sign in and grant access. If you see "app isn't verified", click **Advanced** → **Go to [app name] (unsafe)** — this is expected for personal OAuth apps.

```bash
gws auth login
```

Verify success:

```bash
gws auth status
```

### Build and restart

Clear stale per-group agent-runner copies:

```bash
rm -r data/sessions/*/agent-runner-src 2>/dev/null || true
```

Rebuild the container:

```bash
cd container && ./build.sh
```

Compile and restart:

```bash
npm run build
systemctl --user restart nanoclaw   # Linux
# macOS: launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

## Phase 4: Verify

Tell the user:

> Google Workspace is connected! Try sending this in your main channel:
>
> `@Claude list my recent emails` or `@Claude what's on my calendar today?`

Check logs if needed:

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### `gws: command not found` inside container

The CLI must be installed globally in the container image. Rebuild after installing:

```bash
cd container && ./build.sh
```

### Token expired / auth error

Re-authorize:

```bash
gws auth logout
gws auth login
```

### Permission denied on mount

Ensure `~/.config/gws` exists and has correct permissions:

```bash
ls -la ~/.config/gws/
chmod 700 ~/.config/gws/
```

### Container can't access gws tools

- Verify mount: check `src/container-runner.ts` for the `gwsConfigDir` block
- Check `mcp__gws__*` is in `allowedTools` in `container/agent-runner/src/index.ts`
- Clear stale copies: `rm -r data/sessions/*/agent-runner-src 2>/dev/null || true`

## Removal

1. Remove the `gwsConfigDir` mount block from `src/container-runner.ts`
2. Remove `'mcp__gws__*'` from `allowedTools` in `container/agent-runner/src/index.ts`
3. Remove the `gws` entry from `mcpServers` in `container/agent-runner/src/index.ts`
4. Clear stale agent-runner copies: `rm -r data/sessions/*/agent-runner-src 2>/dev/null || true`
5. Rebuild and restart:
   ```bash
   cd container && ./build.sh && cd .. && npm run build
   systemctl --user restart nanoclaw   # Linux
   # macOS: launchctl kickstart -k gui/$(id -u)/com.nanoclaw
   ```
6. Optionally uninstall CLI: `npm uninstall -g @googleworkspace/cli`
7. Optionally revoke access: `gws auth logout`
