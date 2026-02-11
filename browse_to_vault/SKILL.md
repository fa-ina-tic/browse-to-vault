---
name: browsing-to-brain
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
      "primaryEnv": "VAULT_DIR"
    }
  }
---

# Browsing-to-Brain Pipeline

## Overview
This skill converts the user's recent Chrome browser history into a connected
Obsidian knowledge base, automatically linking new knowledge to existing notes.

## Vault Location
Use $VAULT_DIR
It can be a local obsidian vault path, or a link to github repository.
If it is a github repository, clone it in local and change $VAULT_DIR to `<GIT_CLONE_DIR>/content`

## Pipeline Steps

### 1. Extract Browser History (last hour)
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

### 2. Filter Entries
Ignore these domains: google.com, youtube.com, github.com, localhost,
chrome://, newtab, extensions, internal pages.
Also skip any URL already processed (check `$VAULT_DIR/.processed_urls`).

### 3. Fetch Page Content
For each remaining URL, use `web_fetch` tool to grab readable content.
Truncate to ~3000 chars. If fetch fails, skip the entry.

### 4. Analyze & Generate Obsidian Note
For each page, analyze the content and generate a structured note:

**Frontmatter:**
```yaml
---
title: "Descriptive Title"
date: <ISO timestamp of visit>
source: <URL>
tags: [relevant, tags, here]
auto_generated: true
---
```

**Body structure:**
- `# Title` — clear, descriptive
- `## Summary` — 2-3 sentence overview of what the user was learning
- `## Key Takeaways` — the important facts, concepts, code snippets
- `## Connections` — [[wikilinks]] to related existing notes in the vault
- `## Source` — link back to original URL

**Critical: Linking**
Before generating, scan existing .md files in `$VAULT_DIR` to find
related notes. Use `[[Note Name]]` wikilink syntax for connections.
Be generous with links — the goal is a densely connected knowledge graph.

### 5. Write Notes
Save each note to `$VAULT_DIR/auto/YYYY-MM/<Title>.md`.
Create directories as needed.

Also append backlinks to related existing notes if they don't already
reference the new note. Add a line like:
`- Auto-linked: [[new-note-name]]`

### 6. Update Processed URLs
Append each processed URL to `$VAULT_DIR/.processed_urls` (one per line)
to avoid reprocessing.

### 7. Git Commit & Push
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
- Process max 15 entries per run to avoid excessive API usage
- Rate-limit: pause ~2 seconds between page analyses
- If git push fails (network), log the error but don't lose the commits
- The skill combines `exec`, `web_fetch`, `write`, and `edit` tools
- On Windows, ensure `sqlite3` is on PATH (install via `choco install sqlite` or `winget install SQLite.SQLite`)
