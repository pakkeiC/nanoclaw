# Add RSS Digest

Sets up a periodic RSS digest: fetches configured feeds on a schedule, evaluates articles by relevance to the user's interests, and sends the top N most valuable articles as a digest message.

Supports Medium member-only (paywalled) articles via session cookie injection.

Run `/add-rss` in Claude Code to configure.

---

## Phase 1: Pre-flight

Check if already configured:

```bash
ls groups/*/rss-config.json 2>/dev/null && echo "Found" || echo "Not configured"
```

If found, ask the user: are they adding feeds, updating settings, or reconfiguring from scratch?

Check that `src/container-runner.ts` has the Medium cookie injection block:

```bash
grep -q "MEDIUM_SID" src/container-runner.ts && echo "Injection present" || echo "Missing — run npm run build first"
```

If missing, tell the user to re-run this skill after rebuilding (the injection was added in Phase 3 of install).

---

## Phase 2: Collect Configuration

Use AskUserQuestion or conversational prompts to collect:

### 1. Target group
Which group should receive the digest? List available group folders:
```bash
ls groups/
```
Default to the group the user is currently chatting from (usually `whatsapp_main` or `telegram_main`).

### 2. RSS feed URLs
Ask the user to provide feed URLs. Accept a list. Suggest common ones if they are unsure:
- Hacker News: `https://hnrss.org/frontpage`
- Medium tag: `https://medium.com/feed/tag/<topic>`
- Dev.to: `https://dev.to/feed`
- Reddit: `https://www.reddit.com/r/<subreddit>/.rss`
- Any blog that has an RSS/Atom feed

For each URL, ask for an optional short label (e.g. "Hacker News", "AI Research"). If not provided, derive from the domain.

### 3. Interested topics
Ask: "What topics are you most interested in?" (e.g. "AI, system design, TypeScript, startup advice")

These are stored as free-form strings and used by the agent to score article relevance. More specific = better filtering.

Example: `["AI/ML", "distributed systems", "TypeScript", "startup engineering", "open source tools"]`

### 4. Top N articles
How many articles per digest? Default: **5**. Range: 1–20.

### 5. Schedule
When should the digest run? Accept natural language and convert to cron:
- "daily at 9am" → `0 9 * * *`
- "weekdays at 8am" → `0 8 * * 1-5`
- "every Monday at 7am" → `0 7 * * 1`
- "twice daily" → `0 8,18 * * *`

Ask for the user's timezone if not already known (check `groups/{folder}/CLAUDE.md` for context). Convert to UTC for the cron expression, or use the `TIMEZONE` env var already configured in NanoClaw.

### 6. Medium support (optional)
Ask: "Do you want to read Medium member-only articles?"

If yes, explain:
> To access Medium paywalled articles, I need your Medium session cookie (`sid`). Here's how to find it:
>
> 1. Open medium.com in your browser and log in
> 2. Open DevTools (F12) → Application tab → Cookies → medium.com
> 3. Copy the value of the `sid` cookie
> 4. Optionally also copy the `uid` cookie
>
> These cookies expire in ~30 days. You can update them later by editing `.env`.

Store the values in `.env`:
```bash
# Append to .env (check if already present first)
grep -q "MEDIUM_SID" .env || echo "MEDIUM_SID=" >> .env
grep -q "MEDIUM_UID" .env || echo "MEDIUM_UID=" >> .env
```

Then set the values using the Edit tool on `.env`.

---

## Phase 3: Write Config File

Write `groups/{folder}/rss-config.json` with the collected values:

```json
{
  "feeds": [
    { "url": "https://hnrss.org/frontpage", "label": "Hacker News" },
    { "url": "https://medium.com/feed/tag/programming", "label": "Medium Programming" }
  ],
  "topics": ["AI/ML", "distributed systems", "TypeScript"],
  "topN": 5,
  "schedule": "0 9 * * *",
  "mediumEnabled": true
}
```

Fields:
- `feeds`: array of `{ url, label }`
- `topics`: array of interest strings used for relevance scoring
- `topN`: number of articles per digest
- `schedule`: cron expression
- `mediumEnabled`: whether to pass Medium cookies when fetching medium.com URLs

---

## Phase 4: Create Scheduled Task

Use the `mcp__nanoclaw__schedule_task` MCP tool to register the recurring task.

Parameters:
- `group_folder`: the target group folder (e.g. `whatsapp_main`)
- `chat_jid`: the group's JID (look up from `registered_groups` in the DB, or read from `groups/{folder}/CLAUDE.md` context)
- `schedule_type`: `cron`
- `schedule_value`: the cron expression from config
- `context_mode`: `isolated`
- `prompt`: the full RSS agent prompt below (substitute actual values)

### The Scheduled Task Prompt

```
You are running a scheduled RSS digest task.

## Your Config
Read `/workspace/group/rss-config.json`. It contains:
- `feeds`: list of {url, label} RSS/Atom feeds to fetch
- `topics`: list of topics the user cares about (use for relevance scoring)
- `topN`: max articles to include in the digest
- `mediumEnabled`: whether to use Medium session cookies

## Step 1 — Fetch Feeds

For each feed, run this bash command (replace FEED_URL):

```bash
node -e "
const url = process.argv[1];
const headers = { 'User-Agent': 'NanoClaw RSS/1.0', 'Accept': 'application/rss+xml,application/atom+xml,application/xml,text/xml' };
if (url.includes('medium.com') && process.env.MEDIUM_SID) {
  headers['Cookie'] = 'sid=' + process.env.MEDIUM_SID + (process.env.MEDIUM_UID ? '; uid=' + process.env.MEDIUM_UID : '');
}
fetch(url, { headers })
  .then(r => { if (!r.ok) throw new Error('HTTP ' + r.status); return r.text(); })
  .then(xml => {
    const items = [];
    const re = /<(?:item|entry)>([\s\S]*?)<\/(?:item|entry)>/g;
    let m;
    while ((m = re.exec(xml)) !== null) {
      const b = m[1];
      const get = (tag) => { const r2 = new RegExp('<' + tag + '[^>]*>(?:<!\\[CDATA\\[)?([\\s\\S]*?)(?:\\]\\]>)?<\\/' + tag + '>', 'i'); const x = r2.exec(b); return x ? x[1].trim() : ''; };
      const link = get('link') || (b.match(/<link[^>]+href=[\"']([^\"']+)[\"']/i)||[])[1] || '';
      items.push({
        title: get('title').replace(/<[^>]+>/g,''),
        link,
        guid: get('guid') || get('id') || link,
        published: get('pubDate') || get('published') || get('updated') || '',
        summary: get('description').replace(/<[^>]+>/g,'').slice(0,500) || get('summary').replace(/<[^>]+>/g,'').slice(0,500)
      });
    }
    console.log(JSON.stringify(items));
  })
  .catch(e => { console.error('FEED_ERROR:' + e.message); process.exit(0); });
" FEED_URL
```

Collect all fetched articles. Note which feed label each came from. If a feed returns `FEED_ERROR:`, record it as failed but continue.

## Step 2 — Score and Rank

For each fetched article, assign a relevance score (1–10) based on:

1. **Topic match** (most important): Does the title/summary directly relate to any topic in `topics`? Strong match = +4, partial = +2, none = 0
2. **Content quality signals**: Is it a tutorial, deep-dive, case study, or original research? = +3. Is it an opinion piece, list post, or news brief? = +1
3. **Recency**: Published in last 24h = +2, last 72h = +1, older = 0
4. **Novelty**: Does it cover something genuinely new vs. a rehash of known content? Use your judgment = +1

Select the top `topN` articles by score. If fewer new articles exist than `topN`, include all.

## Step 4 — Compose Digest

Format:

```
📰 RSS Digest — {date, e.g. Mon 16 Mar}

Topics: {comma-separated topics list}

1. *{Title}*
   {feed label} · {time ago or date}
   → {2–3 sentence summary explaining what it covers and why it's relevant to the configured topics}
   {url}

2. ...

{if any feeds failed}
⚠️ Could not fetch: {label} ({error})
```

Keep the total under 4000 characters. If over, shorten summaries to 1 sentence each.

If no articles scored above 0: send "📰 No relevant articles in this digest."

For Telegram: use *bold* (single asterisks), avoid markdown headings.
For WhatsApp: use plain text, no markdown.

## Step 5 — Send

Send the digest using `mcp__nanoclaw__send_message`.
```

---

## Phase 5: Build and Restart

```bash
npm run build
```

Then restart NanoClaw to pick up the new `MEDIUM_SID`/`MEDIUM_UID` env injection:

```bash
# Linux (systemd)
systemctl --user restart nanoclaw

# Or if running manually — find the PID and restart
kill $(cat nanoclaw.pid) && node dist/index.js &> /tmp/nanoclaw.log &
```

---

## Phase 6: Verify

Tell the user how to do a quick test:

> To test immediately without waiting for the schedule, I can manually trigger a digest run. Just send the message: **"run the RSS digest now"** from your WhatsApp/Telegram chat, and the agent will execute the full fetch-evaluate-send flow on demand.

Alternatively, to trigger programmatically:
```bash
# Find the task ID
node -e "
import Database from 'better-sqlite3';
const db = new Database('store/messages.db');
console.log(db.prepare('SELECT id, group_folder, schedule_value, next_run FROM scheduled_tasks').all());
db.close();
" --input-type=module
```

What to look for in logs:
- `Container started` — fetch loop began
- `Agent output: 📰 RSS Digest` — digest composed
- `Message sent` — delivered to group
- `FEED_ERROR:` in container logs — a feed failed (expected for broken feeds, not a crash)

---

## Updating Configuration Later

To add/remove feeds or change topics, edit `groups/{folder}/rss-config.json` directly — the next scheduled run picks up the new config automatically. No restart needed.

To update Medium cookies when they expire:
1. Get new `sid` from browser DevTools
2. Edit `.env` and update `MEDIUM_SID=<new_value>`
3. Restart NanoClaw

---

## Security Notes

- `MEDIUM_SID`/`MEDIUM_UID` are read from `.env` on the host, injected as container env vars — the container cannot access `.env` directly (it is shadowed by `/dev/null` in the project mount)
- Feed URLs are user-configured; the agent fetches them with a standard User-Agent header
