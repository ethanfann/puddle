# Raindrop.io Search DSL Reference

Comprehensive reference for the Raindrop.io search query syntax used with `puddle ls` and bulk operations.

## Basic Syntax

Search queries are passed as the first argument to `puddle ls` or via `--search` for bulk operations:

```bash
puddle ls 'your search query'
puddle tag add 'mytag' --search 'your search query'
puddle move 12345 --search 'your search query'
```

Multiple conditions are combined with spaces (implicit AND).

## Tag Filtering

Filter bookmarks by tags using the `#` prefix:

```bash
# Single tag
puddle ls '#react'

# Multiple tags (AND - must have all)
puddle ls '#react #hooks'

# Tag with spaces (use quotes)
puddle ls '#"web development"'
```

## Type Filtering

Filter by bookmark type using `type:`:

| Type | Description |
|------|-------------|
| `link` | Generic links |
| `article` | Articles/blog posts |
| `image` | Images |
| `video` | Videos |
| `document` | PDFs, docs |
| `audio` | Audio files |

```bash
puddle ls 'type:article'
puddle ls 'type:video'
puddle ls '#tutorial type:video'
```

## Date Filtering

Filter by creation date using `created:`:

```bash
# After a date (exclusive)
puddle ls 'created:>2024-01-01'

# Before a date (exclusive)
puddle ls 'created:<2024-06-01'

# Date range
puddle ls 'created:>2024-01-01 created:<2024-06-01'

# Specific date
puddle ls 'created:2024-03-15'
```

### Dynamic Dates with Shell Substitution

Use shell command substitution for relative dates:

```bash
# Last 7 days
puddle ls "created:>$(date -d '7 days ago' +%Y-%m-%d)"

# Last 30 days
puddle ls "created:>$(date -d '30 days ago' +%Y-%m-%d)"

# This month
puddle ls "created:>$(date +%Y-%m-01)"

# This year
puddle ls "created:>$(date +%Y-01-01)"

# Last week (Monday to Sunday)
puddle ls "created:>$(date -d 'last monday' +%Y-%m-%d)"

# macOS date syntax (BSD date)
puddle ls "created:>$(date -v-7d +%Y-%m-%d)"
puddle ls "created:>$(date -v-1m +%Y-%m-%d)"
```

## Field Filtering

Search within specific fields:

| Field | Description |
|-------|-------------|
| `title:` | Search in bookmark title |
| `link:` | Search in URL |
| `domain:` | Filter by domain |
| `excerpt:` | Search in description |
| `note:` | Search in notes |

```bash
# By domain
puddle ls 'domain:github.com'
puddle ls 'domain:medium.com'

# In title
puddle ls 'title:tutorial'
puddle ls 'title:"getting started"'

# In URL
puddle ls 'link:api'
puddle ls 'link:/docs/'

# In description
puddle ls 'excerpt:introduction'
```

## Special Filters

### Untagged Bookmarks
```bash
puddle ls 'notag:true'
```

### Broken Links
```bash
puddle ls 'broken:true'
```

### Duplicates
```bash
puddle ls 'duplicate:true'
```

### Important/Favorited
```bash
puddle ls 'important:true'
```

### Cached/Permanent Copy
```bash
puddle ls 'cache:true'
```

## Text Search

### Exact Phrase
```bash
puddle ls '"react hooks"'
puddle ls '"getting started with"'
```

### Plain Text (searches title, excerpt, URL)
```bash
puddle ls 'kubernetes deployment'
```

## Boolean Operations

Conditions separated by spaces are ANDed together:

```bash
# Must have both tags AND be an article
puddle ls '#react #typescript type:article'

# Must be from github AND created after date
puddle ls 'domain:github.com created:>2024-01-01'
```

**Note:** Raindrop.io search doesn't support explicit OR or NOT operators. Use multiple queries or post-process with jq.

## Complex Query Examples

### Find old untagged bookmarks
```bash
puddle ls "notag:true created:<$(date -d '1 year ago' +%Y-%m-%d)"
```

### Find recent articles about a topic
```bash
puddle ls "#javascript type:article created:>$(date -d '30 days ago' +%Y-%m-%d)"
```

### Find videos from specific domain
```bash
puddle ls 'type:video domain:youtube.com'
```

### Find bookmarks with specific word in title from this year
```bash
puddle ls "title:tutorial created:>$(date +%Y-01-01)"
```

### Find important bookmarks by tag
```bash
puddle ls '#reference important:true'
```

### Find broken links in a specific domain
```bash
puddle ls 'broken:true domain:example.com'
```

## jq Output Processing

### Extract specific fields
```bash
# URLs only
puddle ls '#react' | jq -r '.[].link'

# Title and URL
puddle ls 'type:article' | jq -r '.[] | "\(.title)\n  \(.link)\n"'

# As CSV
puddle ls '#project' | jq -r '.[] | [._id, .title, .link] | @csv'
```

### Count results
```bash
puddle ls '#react' | jq 'length'
```

### Filter locally (when API filtering is insufficient)
```bash
# Bookmarks with more than 3 tags
puddle ls | jq '[.[] | select(.tags | length > 3)]'

# Bookmarks from specific year
puddle ls | jq '[.[] | select(.created | startswith("2024"))]'

# Bookmarks with specific tag combination (local OR)
puddle ls | jq '[.[] | select(.tags | contains(["react"]) or contains(["vue"]))]'

# Sort by creation date
puddle ls '#project' | jq 'sort_by(.created) | reverse'

# Group by domain
puddle ls | jq 'group_by(.domain) | map({domain: .[0].domain, count: length})'
```

### Combine with bulk operations
```bash
# Tag all bookmarks from a specific domain
domain="old-blog.com"
puddle ls "domain:$domain" | jq -r '.[].id' | while read id; do
  puddle update "$id" --tags 'archived,old-blog'
done

# Or use bulk tag (more efficient)
puddle tag add 'archived' --search "domain:$domain"
```

### Export bookmarks
```bash
# Export to JSON file
puddle ls '#export' | jq '.' > export.json

# Export as markdown links
puddle ls '#reference' | jq -r '.[] | "- [\(.title)](\(.link))"'

# Export as HTML bookmarks
puddle ls '#backup' | jq -r '.[] | "<a href=\"\(.link)\">\(.title)</a><br>"'
```

## Pagination for Large Results

```bash
# Get all results with pagination
page=0
while true; do
  result=$(puddle ls '#project' --page "$page" --perpage 50)
  count=$(echo "$result" | jq 'length')
  echo "$result"
  [[ "$count" -lt 50 ]] && break
  ((page++))
done
```

## Collection Scoping

Limit searches to specific collections:

```bash
# Search within collection
puddle ls '#react' --collection 12345

# Get collection ID first
puddle collections | jq '.[] | select(.title == "Development") | ._id'
```

## Tips

1. **Quote complex queries** to prevent shell interpretation
2. **Use double quotes** for shell variable expansion in dates
3. **Use single quotes** for literal strings with special characters
4. **Combine with jq** for complex filtering the API doesn't support
5. **Use bulk operations** (`tag`, `move`) instead of loops when possible
