---
name: puddle
description: CLI for Raindrop.io bookmark management - search, create, update, delete, bulk operations. Use when working with Raindrop.io bookmarks or when the user mentions bookmarks, link saving, or Raindrop.
license: MIT
compatibility: Requires bash 4+, curl, jq
---

# puddle

Bash CLI for managing Raindrop.io bookmarks. Designed for AI agents to programmatically manage bookmarks.

## Setup

```bash
# Set token (get from https://app.raindrop.io/settings/integrations)
export RAINDROP_TOKEN="your-token"

# Or save to config file
mkdir -p ~/.config/puddle
echo "your-token" > ~/.config/puddle/token
```

## Commands

| Command | Description |
|---------|-------------|
| `puddle ls [search]` | List/search bookmarks |
| `puddle get <id>` | Get single bookmark |
| `puddle add <url>` | Create bookmark |
| `puddle update <id>` | Update bookmark |
| `puddle rm <id>` | Delete bookmark |
| `puddle tag add\|rm <tag> --search '...'` | Bulk tag operations |
| `puddle move <collection-id> --search '...'` | Bulk move bookmarks |
| `puddle collections` | List all collections |

## Quick Examples

```bash
# Search by tag
puddle ls '#react'

# Search by type
puddle ls 'type:article'

# Add bookmark with metadata
puddle add 'https://example.com' --title 'Example' --tags 'dev,reference' --collection 12345

# Update bookmark
puddle update 12345 --tags 'updated,tags' --important true

# Bulk tag all matching bookmarks
puddle tag add 'archived' --search 'created:<2024-01-01'

# Move bookmarks to collection
puddle move 67890 --search '#old-project'

# Get collection IDs
puddle collections | jq '.[] | {id: ._id, title}'
```

## Command Options

### `ls [search]`
```
--collection <id>    Filter by collection (default: all)
--page <n>           Page number (default: 0)
--perpage <n>        Results per page (default: 25)
```

### `add <url>`
```
--title <text>       Override auto-detected title
--tags <a,b,c>       Comma-separated tags
--collection <id>    Target collection
--excerpt <text>     Description/excerpt
```

### `update <id>`
```
--title <text>       New title
--note <text>        Agent note field
--tags <a,b,c>       Replace all tags
--excerpt <text>     New description
--collection <id>    Move to collection
--important true|false   Mark as favorite
```

### `tag add|rm <tag>`
```
--search <query>     Required: search query for matching bookmarks
--collection <id>    Limit to collection (default: all)
```

### `move <target-collection-id>`
```
--search <query>     Required: search query for matching bookmarks
--collection <id>    Source collection (default: all)
```

## Search Syntax Quick Reference

| Syntax | Example | Description |
|--------|---------|-------------|
| `#tag` | `#react` | Filter by tag |
| `type:X` | `type:article` | Filter by type (article, video, image, document, audio, link) |
| `created:>DATE` | `created:>2024-01-01` | Created after date |
| `created:<DATE` | `created:<2024-06-01` | Created before date |
| `domain:X` | `domain:github.com` | Filter by domain |
| `title:X` | `title:tutorial` | Search in title |
| `link:X` | `link:api` | Search in URL |
| `notag:true` | `notag:true` | Bookmarks without tags |
| `"exact phrase"` | `"react hooks"` | Exact phrase match |

Combine with spaces: `#react type:article created:>2024-01-01`

For complex queries, see [references/search-dsl.md](references/search-dsl.md).

## Output Processing with jq

```bash
# Extract URLs only
puddle ls '#react' | jq -r '.[].link'

# Get titles and IDs
puddle ls 'type:video' | jq '.[] | {id: ._id, title}'

# Count results
puddle ls '#project' | jq 'length'

# Filter locally
puddle ls | jq '[.[] | select(.tags | contains(["important"]))]'
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error (missing token, invalid args, network) |
| 2 | API error (4xx response) |
| 3 | Not found / Invalid ID |

## Global Options

```
--token <token>    Override token (useful for scripts)
--json             Output errors as JSON (for programmatic parsing)
--version          Show version
--help             Show help
```
