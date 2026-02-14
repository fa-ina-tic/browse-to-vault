---
name: browse-to-vault
description: >
  Hourly pipeline that extracts Chrome browser history,
  analyzes visited pages with AI, generates Obsidian-formatted
  markdown notes with [[wikilinks]] to existing notes, and
  pushes changes to the sub-brain GitHub repo.
metadata: {
  "openclaw": {
      "requires": {
        "bins": ["sqlite3", "git"],
        "config": ["browser.enabled"]
      },
      "primaryEnv": "VAULT_DIR",
      "optionalEnv": "CONTENT_FILTER"
    }
  }
---

# Browsing-to-Brain Pipeline

## Overview
This skill converts the user's recent Chrome browser history into a connected
Obsidian knowledge base, automatically linking new knowledge to existing notes.

## Configuration

### Vault Location
Use `$VAULT_DIR`
It can be a local obsidian vault path, or a link to github repository.
If it is a github repository, clone it in local and change $VAULT_DIR to `<GIT_CLONE_DIR>/content`

### Content Filter (Optional)
Use `$CONTENT_FILTER` to specify what type of content to extract from web pages.

Examples:
- `"tech/cs"` - Extract only technical/computer science information
- `"business"` - Extract business, entrepreneurship, and management content
- `"science"` - Extract scientific research and academic content
- `"design"` - Extract design, UX/UI, and creative content
- `"all"` - Extract all content without filtering (default)

If not set, defaults to `"all"` (no filtering).

## Pipeline Steps

### 1. Initialize SQLite Database
Create/verify the tracking database at `$VAULT_DIR/.browsing_history.db`:

```sql
CREATE TABLE IF NOT EXISTS processed_urls (
  url TEXT PRIMARY KEY,
  processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  note_path TEXT
);
```

This database replaces the old `.processed_urls` text file and provides better
tracking of which URLs have been processed and where their notes are stored.

### 2. Extract Browser History (last hour)
Use `exec` to run a shell command that copies and queries the Chrome
history SQLite database. Chrome locks the DB while running, so always
copy it first.

**Detect OS and set paths:**

macOS:
- History DB: `~/Library/Application Support/Google/Chrome/Default/History`
- Temp copy: `/tmp/history_copy`

Linux:
- History DB: `~/.config/google-chrome/Default/History`
- Temp copy: `/tmp/history_copy`

Windows:
- History DB: `%LOCALAPPDATA%\Google\Chrome\User Data\Default\History`
  (typically `C:\Users\<USERNAME>\AppData\Local\Google\Chrome\User Data\Default\History`)
- Temp copy: `%TEMP%\history_copy`

**Query (same across all platforms):**
```bash
# macOS / Linux
cp "<HISTORY_DB_PATH>" /tmp/history_copy
sqlite3 /tmp/history_copy "
  SELECT url, title, visit_count,
    datetime(last_visit_time/1000000-11644473600, 'unixepoch', 'localtime') as visited
  FROM urls
  WHERE last_visit_time > (strftime('%s','now','-1 hour')+11644473600)*1000000
  ORDER BY last_visit_time DESC;
"
rm /tmp/history_copy
```
```powershell
# Windows (PowerShell)
Copy-Item "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\History" "$env:TEMP\history_copy"
sqlite3 "$env:TEMP\history_copy" "
  SELECT url, title, visit_count,
    datetime(last_visit_time/1000000-11644473600, 'unixepoch', 'localtime') as visited
  FROM urls
  WHERE last_visit_time > (strftime('%s','now','-1 hour')+11644473600)*1000000
  ORDER BY last_visit_time DESC;
"
Remove-Item "$env:TEMP\history_copy"
```

**Chrome Profile Note:** The `Default` folder refers to the default Chrome
profile. If the user uses a named profile, it will be under
`Profile 1`, `Profile 2`, etc. Check `chrome://version` → "Profile Path"
to confirm.

### 3. Filter Entries
Ignore these domains: google.com, youtube.com, github.com, localhost,
chrome://, newtab, extensions, internal pages.

Also skip any URL already processed by querying the SQLite database:
```sql
SELECT url FROM processed_urls WHERE url = '<URL_TO_CHECK>';
```

If the query returns a result, skip that URL.

### 4. Read Page Content
Use the built-in web reading capability (WebFetch / web search agent) to read the URL directly — no curl, no HTML parsing, no shell commands needed.

- Ask the agent to extract the full readable text of the page
- If the page cannot be read or returns no meaningful content, skip the entry

### 5. Analyze & Generate Obsidian Note
For each page, analyze the content and generate a structured note:

**Content Filtering (based on `$CONTENT_FILTER`)**

- If `$CONTENT_FILTER` is set (e.g. `tech/cs`, `business`, `science`): extract ONLY information relevant to that domain — skip unrelated sections
- If `$CONTENT_FILTER` is `all` or unset: extract all significant content

**Focus on extracting skills, insights, and actionable knowledge — not just summaries.**
Go deep on:
- Concepts and mental models introduced
- Techniques, methods, patterns, or workflows explained
- Key facts, data points, or findings
- Concrete examples, code snippets, or case studies
- Opinions or stances the author takes

Avoid: shallow one-line summaries, restating the title, or paraphrasing the intro paragraph only.

**Frontmatter**

```yaml
---
title: "Descriptive Title"
date: <ISO timestamp of visit>
source: <URL>
tags: [relevant, tags, here]
content_type: <value from $CONTENT_FILTER or "general">
auto_generated: true
---
```

**Body Structure**

- `# Title` — clear, descriptive
- `## Summary` — 2–3 sentences: what this page is, why it matters
- `## Key Takeaways` — the actual knowledge: concepts, techniques, insights, examples extracted from the content. Use sub-sections if the content has distinct topics. Be specific and substantive.
- `## Connections` — `[[wikilinks]]` to **genuinely related** existing notes only. Do not link to every note in the vault — only link if there is a real conceptual or practical relationship. Prefer fewer, stronger links over many weak ones.
- `## Source` — link back to the original URL

### 6. Write Notes
Save each note to `$VAULT_DIR/<ContentType>/<Title>.md`.
Create directories as needed.

Also append backlinks to related existing notes if they don't already
reference the new note. Add a line like:
`- Auto-linked: [[new-note-name]]`

### 7. Update Processed URLs Database
For each successfully processed URL, insert it into the SQLite database:

```sql
INSERT INTO processed_urls (url, note_path)
VALUES ('<URL>', '<path/to/generated/note.md>');
```

This tracks both the URL and the location of the generated note for future reference.

### 8. Git Commit & Push
If you are working in git repo, you have to apply changes to the remote repository
```bash
cd ~/sub-brain
git add -A
git diff --cached --quiet || git commit -m "🧠 Auto-update: browsing insights $(date '+%Y-%m-%d %H:%M')"
git push origin main
```

On Windows:
```powershell
Set-Location ~/sub-brain
git add -A
$changes = git diff --cached --quiet; if ($LASTEXITCODE -ne 0) {
  git commit -m "🧠 Auto-update: browsing insights $(Get-Date -Format 'yyyy-MM-dd HH:mm')"
  git push origin main
}
```

## Important Notes
- If no new history entries are found, skip silently (no empty commits)
- Only link to existing notes with a genuine topical overlap. Linking everything to everything dilutes the graph.
- The goal of each note is to make the knowledge *retrievable and usable*, not just to record that a page was visited. Extract the insight, not the summary.
- Process max 15 entries per run to avoid excessive API usage
- Rate-limit: pause ~2 seconds between page analyses
- If git push fails (network), log the error but don't lose the commits
- The skill combines `exec`, `web_fetch`, `write`, and `edit` tools
- On Windows, ensure `sqlite3` is on PATH (install via `choco install sqlite` or `winget install SQLite.SQLite`)
