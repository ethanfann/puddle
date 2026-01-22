# PRD: Puddle - Bash CLI for Raindrop.io API

**Date:** 2025-01-21  
**Version:** 2.0 (Bash rewrite)

---

## Problem Statement

AI agents and developers need programmatic access to Raindrop.io bookmarks. The API is simple REST—no complex state management needed. A lightweight Bash wrapper with JSON output lets agents and users compose commands with standard Unix tools.

### Who is affected?
- **Primary:** AI agents using skills to query/manage bookmarks
- **Secondary:** Developers automating bookmark workflows via pipes

---

## Proposed Solution

Puddle is a Bash CLI that wraps Raindrop.io's REST API. It follows Unix philosophy:
- **Do one thing well:** Each command maps to one API endpoint
- **Text streams:** Output is always JSON, pipe to `jq` for formatting
- **Composable:** Combine with `xargs`, `jq`, `grep`, shell scripts

### Dependencies
- `curl` (HTTP requests)
- `jq` (JSON parsing, optional for users)
- Bash 4+ (or POSIX sh with minor adjustments)

---

## End State

- [ ] `puddle` CLI installable via single script or git clone
- [ ] Token-based auth via env var or config file
- [ ] All core CRUD operations work via simple verbs
- [ ] Search uses Raindrop's native query DSL
- [ ] Output is always valid JSON to stdout
- [ ] Errors go to stderr with non-zero exit codes
- [ ] Agent skill (SKILL.md) teaches AI agents usage patterns

---

## API → CLI Mapping

### V1 Commands (8 commands)

| Command | API Endpoint | Description |
|---------|-------------|-------------|
| `puddle ls [search]` | `GET /raindrops/0?search=` | List/search bookmarks |
| `puddle get <id>` | `GET /raindrop/{id}` | Fetch single bookmark |
| `puddle add <url>` | `POST /raindrop` | Create bookmark |
| `puddle update <id>` | `PUT /raindrop/{id}` | Edit bookmark (title, note, tags) |
| `puddle rm <id>` | `DELETE /raindrop/{id}` | Delete bookmark |
| `puddle tag <add\|rm> <tag>` | `PUT /raindrops/{coll}?search=` | Bulk tag operations |
| `puddle move <collection-id>` | `PUT /raindrops/{coll}?search=` | Move bookmarks to collection |
| `puddle collections` | `GET /collections` + `/childrens` | List all collections |

### V2 Commands (deferred)

| Command | API Endpoint | Description |
|---------|-------------|-------------|
| `puddle tags` | `GET /tags/0` | List all tags |
| `puddle highlights` | `GET /highlights` | List/search highlights |
| `puddle highlights add <id>` | `PUT /raindrop/{id}` | Add highlight to bookmark |

### Search Query DSL (passed directly to API)

Raindrop's `search` parameter supports a powerful DSL:

```bash
puddle ls '#react'                     # by tag
puddle ls '#react #hooks'              # multiple tags (AND)
puddle ls 'type:article'               # by type
puddle ls 'type:video'                 # videos only
puddle ls 'created:>2025-01-01'        # after date
puddle ls 'domain:github.com'          # by domain (not documented but works)
puddle ls '#react type:article'        # combine filters
puddle ls 'match:OR react vue'         # OR search
puddle ls '❤️'                         # favorites
puddle ls 'notag:true'                 # untagged items
```

### Command Options

```
puddle ls [search] [options]
  --collection <id>       Filter by collection ID
  --page <n>              Page number (0-indexed)
  --perpage <n>           Items per page (max 50)
  --sort <field>          -created, created, score, title, -title, domain

puddle get <id>
  (no options, returns full bookmark JSON)

puddle add <url> [options]
  --title "..."           Override auto-detected title
  --tags "tag1,tag2"      Comma-separated tags
  --collection <id>       Collection ID (default: -1 unsorted)
  --excerpt "..."         Description

puddle update <id> [options]
  --title "..."           New title
  --note "..."            Note (agents can use for comments/context)
  --excerpt "..."         Description
  --tags "tag1,tag2"      Replace all tags
  --collection <id>       Move to collection
  --important             Mark as favorite (true/false)

puddle rm <id>
  (moves to Trash; deleting from Trash is permanent)

puddle tag <add|rm> <tag> [options]
  --search "..."          Search query to match bookmarks (required)
  --collection <id>       Limit to collection (default: 0 = all)

puddle move <collection-id> [options]
  --search "..."          Search query to match bookmarks (required)
  --collection <id>       Source collection (default: 0 = all)

puddle collections
  (no options, returns all collections with hierarchy)
```

---

## Interface Specifications

### Output Format

All commands output JSON. Single items return the object directly. Lists return arrays.

```bash
# Single bookmark
puddle get 12345
{
  "_id": 12345,
  "link": "https://example.com",
  "title": "Example",
  "excerpt": "...",
  "tags": ["react", "tutorial"],
  "collection": { "$id": 456 },
  "created": "2025-01-21T10:30:00Z",
  "type": "article"
}

# List (just the items array)
puddle ls '#react' 
[
  { "_id": 12345, "link": "...", ... },
  { "_id": 12346, "link": "...", ... }
]

# Collections
puddle collections
[
  { "_id": 456, "title": "React", "count": 42, "parent": null },
  { "_id": 789, "title": "Hooks", "count": 12, "parent": { "$id": 456 } }
]
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error (auth, network, invalid args) |
| 2 | API error (4xx response) |
| 3 | Not found (404) |

### Error Output

Errors go to stderr as JSON:
```json
{"error": "unauthorized", "message": "Token expired or invalid", "status": 401}
```

---

## Authentication

### Token-based only (v1)

No OAuth flow. Users create a "Test token" in Raindrop.io settings.

```bash
# Option 1: Environment variable (recommended for agents)
export RAINDROP_TOKEN="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Option 2: Config file
mkdir -p ~/.config/puddle
echo "your-token-here" > ~/.config/puddle/token
chmod 600 ~/.config/puddle/token

# Option 3: Pass directly (not recommended)
puddle --token "xxx" ls
```

Token lookup order: `--token` flag → `$RAINDROP_TOKEN` → `~/.config/puddle/token`

---

## File Structure

```
puddle/
├── SKILL.md                  # Agent skill definition (quick reference)
├── README.md                 # Setup instructions
├── references/
│   └── SEARCH-DSL.md         # Complete search syntax reference
└── scripts/
    └── puddle                # Main executable (Bash)
```

---

## Agent Skill (SKILL.md)

An [Agent Skills](https://agentskills.io) compatible skill file for AI agents.

**Design principle:** SKILL.md is a quick reference (~100 lines). The complete search DSL lives in `references/SEARCH-DSL.md` for when agents need to construct complex queries.

```yaml
---
name: puddle
description: >
  Query and manage Raindrop.io bookmarks via CLI. Use when the user asks to
  search bookmarks, save URLs, tag/organize bookmarks, or work with their
  Raindrop.io library. Outputs JSON for easy parsing with jq.
license: MIT
compatibility: Requires curl, jq, and RAINDROP_TOKEN env var
allowed-tools: Bash
---

# Puddle - Raindrop.io CLI

## Commands

| Command | Purpose |
|---------|---------|
| `puddle ls [search]` | List/search bookmarks |
| `puddle get <id>` | Get bookmark details |
| `puddle add <url>` | Save new bookmark |
| `puddle update <id>` | Edit bookmark (title, note, tags) |
| `puddle rm <id>` | Delete bookmark |
| `puddle tag <add\|rm> <tag> --search "..."` | Bulk tag operations |
| `puddle move <collection-id> --search "..."` | Move bookmarks to collection |
| `puddle collections` | List all collections |

## Quick Examples

```bash
# Search by tag
puddle ls '#react'

# Search by content + date
puddle ls "agent skills created:>$(date -d '7 days ago' +%Y-%m-%d)"

# Add bookmark
puddle add "https://example.com" --tags "reference,docs"

# Add a note to a bookmark (agent annotation)
puddle update 12345 --note "Summary: This article covers X, Y, Z. Reviewed 2025-01."

# Delete a bookmark
puddle rm 12345

# Bulk tag matching bookmarks
puddle tag add "react" --search "react hooks"

# List collections (for folder-based workflows)
puddle collections

# Move old bookmarks to archive collection
puddle move 12345 --search "created:<2024-01-01"

# Extract as markdown links
puddle ls '#react' | jq -r '.[] | "- [\(.title)](\(.link))"'
```

## Search Syntax (Quick Reference)

| Pattern | Example |
|---------|---------|
| Tag | `#react` |
| Multiple tags | `#react #hooks` |
| Type | `type:article` |
| Date after | `created:>2025-01-01` |
| Favorites | `❤️` |
| Exclude | `-deprecated` |
| OR search | `match:OR react vue` |

**For complex queries, see `references/SEARCH-DSL.md`**

## Common Agent Tasks

| User says | Command |
|-----------|---------|
| "find my react bookmarks" | `puddle ls '#react'` |
| "bookmarks from last week" | `puddle ls "created:>$(date -d '7 days ago' +%Y-%m-%d)"` |
| "tag all kubernetes articles" | `puddle tag add "k8s" --search "kubernetes type:article"` |
| "show me untagged bookmarks" | `puddle ls 'notag:true'` |
| "save this URL about agents" | `puddle add "https://..." --tags "agents"` |
| "what collections do I have" | `puddle collections` |
| "bookmarks in my React folder" | `puddle ls --collection $(puddle collections \| jq -r '.[] \| select(.title=="React") \| ._id')` |
| "move old stuff to archive" | `puddle move <archive-id> --search "created:<2024-01-01"` |
| "move all videos to Media folder" | `puddle move <media-id> --search "type:video"` |
| "add a summary note to this bookmark" | `puddle update 12345 --note "Summary: ..."` |
| "delete this broken link" | `puddle rm 12345` |
```

---

## Search DSL Reference (references/SEARCH-DSL.md)

A comprehensive reference for agents constructing search queries:

```markdown
# Raindrop Search DSL Reference

This reference covers the complete search query syntax for `puddle ls` and `puddle tag --search`.

## Basic Search

Text searches across title, description, URL, and full page content (Pro only).

```bash
# Single word
puddle ls "kubernetes"

# Multiple words (AND)
puddle ls "react hooks"

# Exact phrase
puddle ls '"react server components"'
```

## Tag Filters

Tags are prefixed with `#`. Multi-word tags use quotes.

```bash
# Single tag
puddle ls '#react'

# Multiple tags (AND - must have all)
puddle ls '#react #tutorial'

# Multi-word tag
puddle ls '#"machine learning"'

# Find untagged
puddle ls 'notag:true'
```

## Type Filters

Filter by content type: `link`, `article`, `image`, `video`, `document`, `audio`

```bash
puddle ls 'type:article'
puddle ls 'type:video'
puddle ls '#react type:article'    # Combine with tag
```

## Date Filters

ISO format dates. Use `<` or `>` for before/after.

```bash
# After specific date
puddle ls 'created:>2025-01-01'

# Before specific date
puddle ls 'created:<2025-01-01'

# Specific month
puddle ls 'created:2025-01'

# Specific year
puddle ls 'created:2025'

# Last updated
puddle ls 'lastUpdate:>2025-01-15'
```

### Dynamic Dates (Shell)

Compute dates dynamically for relative queries:

```bash
# Last 7 days
puddle ls "created:>$(date -d '7 days ago' +%Y-%m-%d)"

# Last 30 days
puddle ls "created:>$(date -d '30 days ago' +%Y-%m-%d)"

# This month
puddle ls "created:>$(date +%Y-%m-01)"

# This year
puddle ls "created:>$(date +%Y-01-01)"

# macOS (BSD date)
puddle ls "created:>$(date -v-7d +%Y-%m-%d)"
```

## Field-Specific Search

Search within specific fields only:

```bash
# Title only
puddle ls 'title:kubernetes'
puddle ls 'title:"getting started"'

# Description only
puddle ls 'excerpt:tutorial'

# Notes only
puddle ls 'note:important'

# URL/domain
puddle ls 'link:github'
puddle ls 'link:"docs.docker.com"'
```

## Boolean Operators

```bash
# OR (either term)
puddle ls 'match:OR react vue'

# NOT (exclude term)
puddle ls 'kubernetes -deprecated'

# NOT with tag
puddle ls '#react -#old'
```

## Special Filters

```bash
# Favorites
puddle ls '❤️'

# Uploaded files
puddle ls 'file:true'

# Has permanent copy (cached)
puddle ls 'cache.status:ready'

# No permanent copy
puddle ls '-cache.status:ready'

# Has reminder
puddle ls 'reminder:true'

# Broken links (Pro)
puddle ls 'broken:true'

# Duplicates (Pro)
puddle ls 'duplicate:true'
```

## Complex Query Examples

### "All React articles saved this month"
```bash
puddle ls "#react type:article created:>$(date +%Y-%m-01)"
```

### "Kubernetes videos, not from YouTube"
```bash
puddle ls '#kubernetes type:video -link:youtube'
```

### "Untagged bookmarks from last week"
```bash
puddle ls "notag:true created:>$(date -d '7 days ago' +%Y-%m-%d)"
```

### "Favorites that are articles about AI"
```bash
puddle ls '❤️ type:article "artificial intelligence"'
```

### "Anything with 'agent' in title, tagged with 'ai' or 'ml'"
```bash
puddle ls 'title:agent match:OR #ai #ml'
```

### "Old tutorials to review (created > 1 year ago)"
```bash
puddle ls "#tutorial created:<$(date -d '1 year ago' +%Y-%m-%d)"
```

## Query Building Pattern

For agents constructing queries programmatically:

1. **Start with content terms**: `kubernetes deployment`
2. **Add tag filters**: `#devops`
3. **Add type filter** if needed: `type:article`
4. **Add date constraint** if needed: `created:>YYYY-MM-DD`
5. **Add exclusions** last: `-deprecated`

Example build-up:
```
kubernetes deployment
kubernetes deployment #devops
kubernetes deployment #devops type:article
kubernetes deployment #devops type:article created:>2024-01-01
kubernetes deployment #devops type:article created:>2024-01-01 -deprecated
```

## Output Processing with jq

```bash
# Just titles
puddle ls '#react' | jq -r '.[].title'

# Just URLs
puddle ls '#react' | jq -r '.[].link'

# Title and URL
puddle ls '#react' | jq -r '.[] | "\(.title): \(.link)"'

# Markdown list
puddle ls '#react' | jq -r '.[] | "- [\(.title)](\(.link))"'

# Count results
puddle ls '#react' | jq 'length'

# Filter by domain in results
puddle ls '#react' | jq '.[] | select(.domain == "github.com")'

# Get IDs for further processing
puddle ls '#react' | jq -r '.[]._id'
```
```

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Lines of Bash | < 300 |
| Time to first query (after install) | < 2 min |
| Command response time | < 500ms |
| AI agent can use after reading skill | Yes |

---

## Non-Goals (v1)

- OAuth flow (use test token)
- Local caching
- Interactive prompts (always non-interactive)
- Colored/formatted output (JSON only, use jq)
- Windows native (use WSL/Git Bash)
- Direct CLI installation to PATH (agents execute from scripts/)

---

## Tasks

### Project Setup [setup]
Initialize project structure and create the main executable shell script.

**Verification:**
- `scripts/puddle` exists and is executable
- Running `./scripts/puddle` without args shows usage help
- Running `./scripts/puddle --version` outputs version string

### Token Authentication [core]
Implement token loading from environment variable or config file.

**Verification:**
- With `RAINDROP_TOKEN` set, commands authenticate successfully
- With token in `~/.config/puddle/token`, commands authenticate successfully
- Token flag `--token` overrides env var and config file
- Missing token shows JSON error to stderr and exits with code 1

### List/Search Command [core]
Implement `puddle ls [search]` to list and search bookmarks.

**Verification:**
- `puddle ls` returns JSON array of bookmarks
- `puddle ls '#react'` filters by tag
- `puddle ls 'type:article'` filters by type
- `puddle ls --collection 12345` filters by collection
- `puddle ls --page 1 --perpage 10` paginates correctly
- Empty results return empty array `[]`

### Get Command [core]
Implement `puddle get <id>` to fetch a single bookmark.

**Verification:**
- `puddle get 12345` returns JSON object with bookmark data
- Invalid ID returns JSON error to stderr, exits with code 3
- Output includes _id, link, title, tags, collection, created fields

### Add Command [write]
Implement `puddle add <url>` to create new bookmarks.

**Verification:**
- `puddle add "https://example.com"` creates bookmark, returns JSON with new _id
- `--title "Custom"` overrides auto-detected title
- `--tags "a,b,c"` adds tags as array
- `--collection 12345` places in specified collection
- `--excerpt "Description"` sets description
- Invalid URL returns error

### Update Command [write]
Implement `puddle update <id>` to edit bookmark fields.

**Verification:**
- `puddle update 12345 --title "New Title"` updates title
- `puddle update 12345 --note "Agent note"` updates note field
- `puddle update 12345 --tags "x,y"` replaces tags
- `puddle update 12345 --excerpt "New desc"` updates description
- `puddle update 12345 --collection 456` moves to collection
- `puddle update 12345 --important true` marks as favorite
- Returns updated bookmark JSON

### Remove Command [write]
Implement `puddle rm <id>` to delete bookmarks.

**Verification:**
- `puddle rm 12345` moves bookmark to Trash
- Returns success JSON with deleted bookmark info
- Deleting from Trash (collection -99) permanently removes
- Invalid ID returns error with code 3

### Bulk Tag Command [bulk]
Implement `puddle tag add|rm <tag> --search "..."` for bulk tagging.

**Verification:**
- `puddle tag add "react" --search "react hooks"` adds tag to matching bookmarks
- `puddle tag rm "old" --search "#old"` removes tag from matching bookmarks
- `--collection 12345` limits operation to collection
- Returns JSON with count of modified bookmarks
- Missing `--search` flag returns error

### Bulk Move Command [bulk]
Implement `puddle move <collection-id> --search "..."` to move bookmarks.

**Verification:**
- `puddle move 12345 --search "type:video"` moves matching bookmarks to collection
- `--collection 456` limits source to specific collection
- Returns JSON with count of moved bookmarks
- Invalid collection ID returns error

### Collections Command [read]
Implement `puddle collections` to list all collections.

**Verification:**
- Returns JSON array of all collections (root and nested)
- Each collection has _id, title, count, parent fields
- Nested collections show parent.$id reference
- System collections (-1 Unsorted, -99 Trash) are not included

### Error Handling [core]
Consistent error handling across all commands.

**Verification:**
- All errors output JSON to stderr: `{"error": "...", "message": "...", "status": N}`
- Network errors exit with code 1
- API 4xx errors exit with code 2
- 404 errors exit with code 3
- Success always exits with code 0

### SKILL.md [docs]
Create Agent Skills compatible skill file.

**Verification:**
- SKILL.md exists with valid YAML frontmatter (name, description, license, compatibility)
- Contains command reference table
- Contains quick examples for common operations
- Contains search syntax quick reference
- Points to references/SEARCH-DSL.md for complex queries

### Search DSL Reference [docs]
Create comprehensive search syntax documentation.

**Verification:**
- references/SEARCH-DSL.md exists
- Documents all search operators (tags, types, dates, fields, boolean)
- Includes dynamic date examples with shell command substitution
- Includes complex query examples
- Includes jq output processing examples

### README [docs]
Create setup and usage documentation.

**Verification:**
- README.md exists
- Documents how to get Raindrop.io API token
- Documents token configuration (env var and config file)
- Documents basic usage examples
- Documents dependencies (curl, jq, bash)

## Context

### Patterns
- Raindrop API base: `https://api.raindrop.io/rest/v1`
- Auth header: `Authorization: Bearer $TOKEN`
- All responses are JSON

### Key Files
- `scripts/puddle` - main executable
- `SKILL.md` - agent skill definition
- `references/SEARCH-DSL.md` - search syntax reference

### Non-Goals
- OAuth flow (use test token)
- Local caching
- Interactive prompts
- Colored output
- Windows native support
- Highlights (v2)
- Tags listing (v2)

---

## Open Questions

| Question | Recommendation |
|----------|----------------|
| Support `--all` to auto-paginate? | v2 - adds complexity |
| Parse custom fields from note? | v2 - let users use jq |
| Bulk operations subcommand? | No - compose with shell |

---

## References

- [Raindrop.io API Documentation](https://developer.raindrop.io/)
- [Raindrop Search Syntax](https://help.raindrop.io/using-search)
- [Agent Skills Specification](https://agentskills.io/specification)
