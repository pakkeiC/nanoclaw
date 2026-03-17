---
name: add-rss
description: Add RSS daily digest skill to NanoClaw. Agents manage feed subscriptions via chat commands and deliver a scheduled daily digest. Supports Medium paywalled articles via cookie-based authentication.
---

# Add RSS Daily Digest

This skill adds RSS feed management and daily digest delivery to NanoClaw. Users subscribe to RSS/Atom feeds via chat commands. A scheduled task fetches those feeds each day, summarises the top items with Claude, and delivers a digest to the group.

Medium paywalled articles are supported via browser cookie upload — no third-party proxy services are used.

## Phase 1: Pre-flight

### Check if already applied

Check if `groups/global/CLAUDE.md` already contains an `## RSS Feed Management` section. If it does, skip to Phase 3 (Setup).

### Ask the user

Use `AskUserQuestion`:

> Which group should receive the daily digest?
> - **main** — deliver to your primary chat group
> - **another group** — specify the folder name (e.g. `telegram_main`, `whatsapp_main`)

Store the answer as `TARGET_GROUP_FOLDER`.

Use `AskUserQuestion`:

> What time should the daily digest be delivered? (e.g. `08:00`, `07:30 AM`)
> Default: `08:00` local time

Convert to cron format (`0 8 * * *` for 08:00). Store as `DIGEST_CRON`.

## Phase 2: Apply Code Changes

### Add medium_cookies.txt to .gitignore

Check if `medium_cookies.txt` is already in `.gitignore`. If not, append:

```
# RSS skill — Medium session cookies (sensitive, never commit)
medium_cookies.txt
```

### Add RSS command instructions to global/CLAUDE.md

Append the following section to `groups/global/CLAUDE.md` (after the last section, before EOF):

```markdown
## RSS Feed Management

Users can manage RSS feed subscriptions and the daily digest with these commands:

- `rss add <url> [label]` — Subscribe to a feed
- `rss remove <url|label>` — Unsubscribe from a feed
- `rss list` — List all subscribed feeds with labels
- `rss fetch` — Deliver an immediate digest right now
- `rss schedule <time>` — Change digest delivery time (e.g. `rss schedule 07:30`)
- `rss pause` / `rss resume` — Pause or resume the daily digest
- `rss medium setup` — Upload Medium cookies for paywalled article access
- `rss medium status` — Show Medium cookie expiry date
- `rss medium clear` — Delete Medium cookies and revert to preview-only

### Feed storage

Feed configuration is stored at `/workspace/group/rss_feeds.json`. Read and write this file directly for all subscription changes. Never mutate the existing object in place — always read the full file, create a new object with your changes, and write the new object back.

Schema:
```json
{
  "version": 1,
  "feeds": [
    {
      "url": "https://example.com/feed.rss",
      "label": "Example Blog",
      "added_at": "2026-03-17T08:00:00Z"
    }
  ],
  "digest": {
    "schedule": "0 8 * * *",
    "max_items_per_feed": 10,
    "task_id": "rss-digest-xxxx"
  },
  "medium": {
    "cookies_file": "medium_cookies.txt",
    "saved_at": "2026-03-17T08:00:00Z",
    "expires_at": "2026-04-16T08:00:00Z",
    "warned": false
  },
  "seen_guids": []
}
```

`seen_guids` tracks article GUIDs/links already delivered so the digest never repeats items. Keep this list trimmed to the last 500 entries.

### Handling `rss add <url> [label]`

1. Validate the URL — fetch it with `WebFetch` and confirm it returns parseable RSS or Atom XML. If not, reply with an error and do NOT save.
2. Read `/workspace/group/rss_feeds.json` (create with defaults if absent).
3. Check for duplicates — if the URL already exists, inform the user and stop.
4. Create a new feeds array with the new entry appended. Write the updated config.
5. If `digest.task_id` is empty or not present in `/workspace/tasks.json`, create a new scheduled task via IPC:
   ```json
   {
     "type": "schedule_task",
     "prompt": "<rss-digest-prompt>",
     "schedule_type": "cron",
     "schedule_value": "<cron from digest.schedule>",
     "context_mode": "isolated"
   }
   ```
   Store the returned task ID in `digest.task_id`.
6. Reply confirming the subscription.

### Handling `rss remove <url|label>`

1. Read `rss_feeds.json`.
2. Find the matching feed by URL or label (case-insensitive).
3. Write a new config with that feed removed.
4. If no feeds remain, cancel the scheduled task via IPC (`cancel_task` with `digest.task_id`), then clear `digest.task_id`.
5. Reply confirming removal.

### Handling `rss medium setup`

1. Prompt the user:
   > Please export your Medium cookies using the *"Get cookies.txt LOCALLY"* browser extension (available for Chrome and Firefox).
   > Export cookies for `medium.com`, then send me the file or paste the contents here.
2. When the user sends the file or pastes content:
   - Validate it contains at least one `medium.com` cookie line.
   - Write the content to `/workspace/group/medium_cookies.txt`.
   - Parse the expiry timestamp from the cookie with the highest expiry date.
   - Update `rss_feeds.json` with:
     ```json
     "medium": {
       "cookies_file": "medium_cookies.txt",
       "saved_at": "<now ISO>",
       "expires_at": "<parsed expiry ISO>",
       "warned": false
     }
     ```
   - Reply: ✓ Medium cookies saved. Access active until `<expires_at date>`.

### Handling `rss medium clear`

1. Delete `/workspace/group/medium_cookies.txt` if it exists.
2. Update `rss_feeds.json` — set `medium` to `null`.
3. Reply confirming removal.

### The digest task prompt

When creating or updating the scheduled digest task, use this prompt (fill in the group JID and folder):

---

You are running the daily RSS digest for this group.

**Step 1 — Load config**
Read `/workspace/group/rss_feeds.json`. If it does not exist or has no feeds, send a message: "No RSS feeds configured. Send `rss add <url>` to subscribe." then stop.

**Step 2 — Check Medium cookies**
If `medium.cookies_file` is set, check if `/workspace/group/medium_cookies.txt` exists and if today's date is before `medium.expires_at`.
- If valid: Medium articles will be fetched with cookies (full content).
- If expired and `medium.warned` is false: note the expiry in the digest footer and set `warned: true` in the config.
- If absent or expired: Medium articles fall back to RSS description text only.

**Step 3 — Fetch feeds**
For each feed in `feeds`:
1. Fetch the RSS/Atom URL using `WebFetch`.
2. Parse items — extract: title, link, description/summary, pubDate, guid.
3. Filter out any item whose guid or link already appears in `seen_guids`.
4. Keep at most `digest.max_items_per_feed` new items per feed (default 10).

**Step 4 — Fetch full content for Medium articles**
For any item where the link contains `medium.com` and Medium cookies are valid:
```bash
curl -s -b /workspace/group/medium_cookies.txt "<article_url>" -o /tmp/medium_article.html
```
Extract the article body text from the HTML (look for `<article>` tag or main content area). Use the extracted text instead of the RSS description. If curl fails or returns empty, fall back to RSS description.

**Step 5 — Summarise**
Group new items by feed. For each item, write a 1–2 sentence summary using the full article text (if available) or the RSS description. Prioritise items most relevant to the user based on content quality and recency.

If no new items were found across all feeds, send: "No new articles today." and stop.

**Step 6 — Format and deliver**
Compose the digest in a clean, readable format suitable for the messaging platform. Example structure:

```
*Daily Digest — 17 Mar 2026*

*Hacker News*
• Article title — one-sentence summary.
• Article title — one-sentence summary.

*Martin Fowler*
• Article title — one-sentence summary.

[Medium cookies expire in 3 days — send `rss medium setup` to refresh]
```

Send the digest via `mcp__nanoclaw__send_message`.

**Step 7 — Update seen_guids**
Read `rss_feeds.json` again (it may have changed). Create a new config with all delivered guids/links appended to `seen_guids`. Trim `seen_guids` to the last 500 entries. Write the updated config.

---
```

## Phase 3: Setup

### Bootstrap initial feeds for the target group

Tell the user:

> RSS is configured. You can now add feeds from `TARGET_GROUP_FOLDER` with:
>
> `@Andy rss add <url> [optional label]`
>
> Would you like to add some starter feeds now? If so, share the URLs.

If the user provides URLs, write the initial `rss_feeds.json` to `groups/TARGET_GROUP_FOLDER/rss_feeds.json`:

```json
{
  "version": 1,
  "feeds": [],
  "digest": {
    "schedule": "DIGEST_CRON",
    "max_items_per_feed": 10,
    "task_id": ""
  },
  "medium": null,
  "seen_guids": []
}
```

Then for each URL provided, validate and add it as per the `rss add` command flow. The scheduled task will be created automatically when the first feed is added.

### Optional: Medium cookie setup

Tell the user:

> If you have a Medium subscription and want full article access, send:
>
> `@Andy rss medium setup`
>
> in your chat. I'll walk you through exporting and uploading your cookies.

## Phase 4: Verify

### Confirm configuration

Check that `groups/TARGET_GROUP_FOLDER/rss_feeds.json` exists and has at least one feed:

```bash
cat groups/TARGET_GROUP_FOLDER/rss_feeds.json
```

### Test immediate digest

Tell the user:

> Send this in your chat to trigger an immediate digest:
>
> `@Andy rss fetch`

Monitor logs to confirm the agent runs and delivers a digest:

```bash
tail -f logs/nanoclaw.log | grep -iE "(rss|digest|feed)"
```

### Verify scheduled task

Once the first feed is added via chat, confirm the task was created:

```bash
sqlite3 store/messages.db "SELECT id, schedule_type, schedule_value, status FROM scheduled_tasks WHERE group_folder = 'TARGET_GROUP_FOLDER';"
```

## Troubleshooting

### Digest not firing

- Check the scheduled task exists: `sqlite3 store/messages.db "SELECT * FROM scheduled_tasks;"`
- Check logs: `tail -f logs/nanoclaw.log`
- Ensure `status = 'active'` and `next_run` is in the future

### Medium articles showing preview only

- Check cookies file exists: `ls -la groups/TARGET_GROUP_FOLDER/medium_cookies.txt`
- Check expiry: `cat groups/TARGET_GROUP_FOLDER/rss_feeds.json | grep expires_at`
- Re-upload via `@Andy rss medium setup`

### Feed not parsing

- Test fetch manually: `curl -s "<feed_url>" | head -50`
- Confirm the URL returns RSS or Atom XML (look for `<rss>` or `<feed>` root element)
- Some feeds require `Accept` headers — note this in the feed's label for reference

## Removal

1. Remove the `## RSS Feed Management` section from `groups/global/CLAUDE.md`
2. Cancel the digest scheduled task:
   ```bash
   sqlite3 store/messages.db "UPDATE scheduled_tasks SET status = 'completed' WHERE id LIKE 'rss-digest-%';"
   ```
3. Remove feed configs:
   ```bash
   rm -f groups/*/rss_feeds.json groups/*/medium_cookies.txt
   ```
4. Remove the `medium_cookies.txt` line from `.gitignore`
