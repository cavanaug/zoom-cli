# zoom-cli Design Specification

**Version**: 1.0  
**Last Updated**: 2026-02-23

## Overview

This document describes the **user-facing design** of `zoom-cli`, including command usage, syntax, configuration, and workflows.

For implementation details, see:
- **[IMPLEMENTATION.md](./IMPLEMENTATION.md)** - Technology stack, project structure, and development guide
- **[API-REFERENCE.md](./API-REFERENCE.md)** - Zoom API integration details and OAuth implementation

---

## Table of Contents

1. [Design Philosophy](#design-philosophy)
2. [Virtual Filesystem Structure](#virtual-filesystem-structure)
3. [Core Commands](#core-commands)
4. [Path Syntax](#path-syntax)
5. [Date and Time Syntax](#date-and-time-syntax)
6. [Configuration](#configuration)
7. [Output Formats](#output-formats)
8. [Error Handling](#error-handling)
9. [Example Workflows](#example-workflows)

---

## Design Philosophy

### Filesystem Metaphor

`zoom-cli` treats Zoom meetings as a virtual filesystem where:
- **Meeting series** are top-level directories (e.g., `/Team Standup/`)
- **Meeting instances** are date-based subdirectories (e.g., `/Team Standup/2026-02-23/`)
- **Resources** are files with extensions (e.g., `summary.md`, `transcript.vtt`, `recording.mp4`)

### Core Principles

1. **Stateless**: All commands use explicit paths. No context switching.
2. **Explicit**: File extensions are required. No guessing or magic defaults.
3. **Scriptable**: Designed for both interactive use and automation.
4. **Unix-like**: Familiar commands (`ls`, `cat`, `cp`) with standard semantics.
5. **Smart Defaults**: Sensible defaults for common operations, configurable for power users.

---

## Virtual Filesystem Structure

```
/ (root)
├── Team-Standup/                    # Meeting series (directory)
│   ├── @latest/                     # Alias: most recent instance
│   ├── @1/                          # Alias: most recent instance (same as @latest)
│   ├── @2/                          # Alias: 2nd most recent instance
│   ├── @3/                          # Alias: 3rd most recent instance
│   ├── 2026-02-23/                  # Meeting instance by date (YYYY-MM-DD)
│   │   ├── summary.md               # AI-generated meeting summary (Markdown)
│   │   ├── transcript.vtt           # Meeting transcript (WebVTT format)
│   │   ├── transcript.srt           # Meeting transcript (SubRip format)
│   │   ├── recording.mp4            # Video recording
│   │   ├── recording.m4a            # Audio-only recording
│   │   ├── chat.txt                 # Chat log (if available)
│   │   └── metadata.json            # Meeting metadata (participants, duration, etc.)
│   ├── 2026-02-22/
│   │   ├── summary.md
│   │   └── transcript.vtt
│   └── 2026-02-21/
│       └── summary.md
│
├── Client-Review/                   # Another meeting series
│   ├── @latest/
│   ├── 2026-02-20/
│   └── 2026-02-13/
│
└── 1on1-with-Manager/               # Meeting series (normalized name)
    └── 2026-02-19/
```

### Path Components

- **Root** (`/`): Lists all meeting series
- **Meeting Series** (`/Team-Standup/`): A recurring meeting or collection of related meetings
- **Meeting Instance** (`/Team-Standup/2026-02-23/`): A specific occurrence of a meeting
- **Resource** (`/Team-Standup/2026-02-23/summary.md`): A file associated with a meeting

### Special Aliases

- `@latest` or `@1`: Most recent meeting instance
- `@2`, `@3`, `@N`: Nth most recent instance (1-indexed)

### Name Normalization

Meeting series names are normalized for filesystem compatibility:
- Spaces replaced with hyphens: `Team Standup` → `Team-Standup`
- Special characters removed: `1:1 Meeting!` → `1on1-Meeting`
- Case preserved: `Team Standup` → `Team-Standup` (not `team-standup`)
- Multiple consecutive spaces/special chars collapsed: `Team  Standup` → `Team-Standup`

---

## Core Commands

### `ls` - List Directory Contents

**Purpose**: Browse the virtual filesystem

**Syntax**:
```bash
zoom-cli ls [PATH] [FLAGS]
```

**Examples**:
```bash
# List all meeting series
zoom-cli ls /

# List instances of a specific series
zoom-cli ls "/Team-Standup/"

# List resources for a specific instance
zoom-cli ls "/Team-Standup/@latest/"
zoom-cli ls "/Team-Standup/2026-02-23/"

# Glob patterns
zoom-cli ls /Team*/                      # All series starting with "Team"
zoom-cli ls /*/*/                        # All instances across all series
zoom-cli ls "/Team-Standup/*/summary.md" # All summaries for Team Standup
```

**Flags**:
- `-l, --long`: Detailed list view with metadata
- `--since <date>`: Filter instances since date (see Date Syntax)
- `--output <format>`: Output format (table, json, csv)

**Output Examples**:

*Default (simple list)*:
```
Team-Standup
Client-Review
1on1-with-Manager
```

*Detailed view (`-l`)*:
```
15 instances  Team-Standup        Last: 2026-02-23 10:30
 8 instances  Client-Review       Last: 2026-02-20 14:00
12 instances  1on1-with-Manager   Last: 2026-02-19 09:00
```

*Instance listing*:
```bash
zoom-cli ls "/Team-Standup/" -l
```
```
DATE        TIME   DURATION  PARTICIPANTS  RESOURCES
2026-02-23  10:30  30m       5             summary.md, transcript.vtt
2026-02-22  10:30  28m       4             summary.md, transcript.vtt
2026-02-21  10:30  32m       5             summary.md, transcript.vtt, recording.mp4
```

---

### `cat` - View Resource Content

**Purpose**: Display the contents of a resource

**Syntax**:
```bash
zoom-cli cat <PATH> [FLAGS]
```

**Examples**:
```bash
# View summary
zoom-cli cat "/Team-Standup/@latest/summary.md"

# View transcript
zoom-cli cat "/Team-Standup/@latest/transcript.vtt"

# View metadata
zoom-cli cat "/Team-Standup/@latest/metadata.json"

# View specific instance
zoom-cli cat "/Team-Standup/2026-02-20/summary.md"
```

**Flags**:
- `--output <format>`: Override output format
- `--raw`: Display raw content without formatting

**Behavior**:
- Markdown files are displayed with basic formatting in terminal (if possible)
- JSON files are pretty-printed
- Large files are automatically piped through a pager
- Binary files (recordings) display an error message

---

### `cp` - Copy Resources

**Purpose**: Copy resources from Zoom to local filesystem using standard `cp` semantics

**Syntax**:
```bash
zoom-cli cp [FLAGS] <SOURCE> <DEST>
```

**Examples**:
```bash
# Copy single file
zoom-cli cp "/Team-Standup/@latest/summary.md" ~/Downloads/

# Copy entire instance (all available resources)
zoom-cli cp -r "/Team-Standup/@latest/" ~/Downloads/team-standup-latest/

# Copy specific resource across all instances
zoom-cli cp "/Team-Standup/*/summary.md" ~/summaries/
```

**Flags**:
- `-r, --recursive`: Copy directories recursively

**Behavior**:
- Follows standard `cp` semantics
- Preserves exact directory structure when using `-r`
- Copies **all** available resources (including recordings)
- No filtering or smart defaults - explicit is better than implicit
- If destination is a directory, files are copied into it
- If destination doesn't exist, creates necessary directories

**Directory Structure Preservation**:
```bash
zoom-cli cp -r "/Team-Standup/@latest/" ~/Downloads/
```
Creates:
```
~/Downloads/
└── 2026-02-23/
    ├── summary.md
    ├── transcript.vtt
    ├── recording.mp4
    └── metadata.json
```

---

### `download` - Smart Download with Filtering

**Purpose**: Download resources with smart defaults and configuration support

**Syntax**:
```bash
zoom-cli download [FLAGS] <SOURCE> <DEST>
```

**Examples**:
```bash
# Download with default filters (excludes recordings)
zoom-cli download "/Team-Standup/@latest/" ~/Downloads/

# Download multiple instances with date prefix
zoom-cli download "/Team-Standup/*/" ~/summaries/

# Preserve structure
zoom-cli download "/Team-Standup/*/" ~/summaries/ --preserve-structure

# Override config to include recordings
zoom-cli download "/Team-Standup/@latest/" ~/Downloads/ --include recordings

# Download since date
zoom-cli download "/Team-Standup/*/" ~/summaries/ --since 7d
```

**Flags**:
- `--since <date>`: Only download instances since date
- `--preserve-structure`: Keep full directory hierarchy
- `--include <resources>`: Comma-separated list of additional resources to include
- `--exclude <resources>`: Comma-separated list of resources to exclude
- `--dry-run`: Show what would be downloaded without downloading

**Default Behavior** (configurable via `config.yaml`):
- Downloads: `summary.md`, `transcript.vtt`, `metadata.json`
- Excludes: `recording.mp4`, `recording.m4a` (large files)
- File organization: Date-prefixed flat structure

**File Organization** (without `--preserve-structure`):
```bash
zoom-cli download "/Team-Standup/*/" ~/summaries/
```
Creates:
```
~/summaries/
├── 2026-02-23-summary.md
├── 2026-02-23-transcript.vtt
├── 2026-02-23-metadata.json
├── 2026-02-22-summary.md
├── 2026-02-22-transcript.vtt
└── 2026-02-22-metadata.json
```

**With `--preserve-structure`**:
```bash
zoom-cli download "/Team-Standup/*/" ~/summaries/ --preserve-structure
```
Creates:
```
~/summaries/
└── Team-Standup/
    ├── 2026-02-23/
    │   ├── summary.md
    │   ├── transcript.vtt
    │   └── metadata.json
    └── 2026-02-22/
        ├── summary.md
        ├── transcript.vtt
        └── metadata.json
```

---

### `auth` - Authentication Management

**Purpose**: Manage OAuth authentication

**Subcommands**:

#### `auth login`
Initiate OAuth flow and store credentials.

```bash
zoom-cli auth login
```

**Behavior**:
1. Opens browser to Zoom OAuth authorization page
2. Generates PKCE and CSRF protection values (`code_verifier`, `code_challenge`, `state`)
3. Selects first available configured callback URL and starts local callback server
4. User authorizes the application
5. Validates callback `state` and exchanges authorization code for access token + refresh token
6. Stores tokens securely in `~/.config/zoom-cli/tokens.json` (0600 permissions)
7. Displays success message with user info

**Port behavior**:
- Redirect URIs must match values configured in Zoom Marketplace.
- `auth login` tries configured callback URLs in order and uses the first bindable port.
- If all configured callback ports are in use, the command exits with an actionable error.

#### `auth logout`
Clear stored authentication tokens.

```bash
zoom-cli auth logout
```

#### `auth status`
Display current authentication status.

```bash
zoom-cli auth status
```

**Output**:
```
Status: Authenticated
User: john@example.com
Scopes: meeting:read, meeting_summary:read, recording:read
Token expires: 2026-02-24 10:30:00
Refresh token: valid
```

#### `auth whoami`
Display current user information.

```bash
zoom-cli auth whoami
```

**Output**:
```
Name: John Doe
Email: john@example.com
Account Type: Pro
Timezone: America/New_York
```

---

### `config` - Configuration Management

**Purpose**: Manage CLI configuration

**Subcommands**:

#### `config list`
Display all configuration values.

```bash
zoom-cli config list
```

#### `config get <key>`
Get a specific configuration value.

```bash
zoom-cli config get defaults.output_format
zoom-cli config get download.default_resources
```

#### `config set <key> <value>`
Set a configuration value.

```bash
zoom-cli config set defaults.output_format json
zoom-cli config set download.default_resources "summary.md,transcript.vtt"
```

#### `config edit`
Open config file in default editor.

```bash
zoom-cli config edit
```

#### `config help [section]`
Show help for configuration options.

```bash
zoom-cli config help
zoom-cli config help download
zoom-cli config help auth
```

#### `config init`
Generate a default configuration file template.

```bash
zoom-cli config init
```

---

## Path Syntax

### Quoted Paths

Paths with spaces can be quoted:
```bash
zoom-cli ls "/Team Standup/"
zoom-cli cat "/Team Standup/@latest/summary.md"
```

### Escaped Paths

Or use backslash escaping:
```bash
zoom-cli ls /Team\ Standup/
zoom-cli cat /Team\ Standup/@latest/summary.md
```

### Fuzzy Matching

When unambiguous, partial names are matched:
```bash
zoom-cli ls /standup/          # Matches /Team-Standup/ if unique
zoom-cli ls /team/             # Error if multiple matches (Team-Standup, Team-Planning)
```

### Glob Patterns

Standard glob patterns are supported:

- `*` - Match zero or more characters within a path component
- `?` - Match single character
- `[abc]` - Match any character in set
- `[a-z]` - Match any character in range

**Examples**:
```bash
zoom-cli ls /Team*/                      # All series starting with "Team"
zoom-cli ls /*/*/summary.md              # All latest summaries
zoom-cli ls "/Team-Standup/2026-02-*/summary.md"  # All summaries from February 2026
```

**Important**: Globs do **not** expand recursively across directory levels unless explicitly specified:
- `/*/summary.md` - ERROR (instances needed: `/*/*/summary.md`)
- `/*/*/summary.md` - Correct (series/instance/file)

---

## Date and Time Syntax

Based on [Taskwarrior date syntax](https://taskwarrior.org/docs/dates/).

### Absolute Dates

**ISO-8601 format**:
- `2026-02-23` - Specific date
- `2026-02` - Entire month (implies 2026-02-01 to 2026-02-29)
- `2026` - Entire year

### Relative Dates

**Time periods**:
- `7d` - 7 days ago
- `2w` - 2 weeks ago
- `1m` - 1 month ago
- `3y` - 3 years ago

**Named dates**:
- `today` - Current date (00:00:00)
- `yesterday` - Previous day (00:00:00)
- `tomorrow` - Next day (00:00:00)
- `now` - Current date and time

**Week boundaries**:
- `sow` - Start of week (Monday 00:00:00)
- `eow` - End of week (Sunday 23:59:59)
- `soww` - Start of work week (Monday 00:00:00)
- `eoww` - End of work week (Friday 23:59:59)

**Month boundaries**:
- `som` - Start of month (1st day, 00:00:00)
- `eom` - End of month (last day, 23:59:59)

**Quarter boundaries**:
- `soq` - Start of quarter (00:00:00)
- `eoq` - End of quarter (23:59:59)

**Year boundaries**:
- `soy` - Start of year (January 1st, 00:00:00)
- `eoy` - End of year (December 31st, 23:59:59)

**Day names**:
- `monday`, `tuesday`, ..., `sunday` - Next occurrence of that day
- `mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun` - Abbreviated forms

### Special Aliases

- `@latest` - Most recent meeting instance (same as `@1`)
- `@1`, `@2`, `@3`, ... - Nth most recent instance
- `@last-run` - Timestamp from last sync operation (future: sync feature)

### Usage in Commands

```bash
# List meetings since 7 days ago
zoom-cli ls "/Team-Standup/" --since 7d

# List meetings from this week
zoom-cli ls "/Team-Standup/" --since sow

# Download summaries from last month
zoom-cli download "/Team-Standup/*/" ~/summaries/ --since 1m

# List meetings from specific date
zoom-cli ls "/Team-Standup/" --since 2026-02-01
```

---

## Configuration

### File Location

```
~/.config/zoom-cli/config.yaml
```

Or use `XDG_CONFIG_HOME`:
```
$XDG_CONFIG_HOME/zoom-cli/config.yaml
```

### Minimal Configuration

Only OAuth client ID and redirect URL are required for PKCE-first setup:

```yaml
auth:
  client_id: "your-zoom-client-id"
  redirect_url: "http://localhost:53682/callback"
  redirect_urls:
    - "http://localhost:53682/callback"
    - "http://localhost:53683/callback"
    - "http://localhost:53684/callback"
```

If your specific app configuration requires confidential token exchange, include `client_secret`:

```yaml
auth:
  client_id: "your-zoom-client-id"
  client_secret: "your-zoom-client-secret"
  redirect_url: "http://localhost:53682/callback"
  redirect_urls:
    - "http://localhost:53682/callback"
    - "http://localhost:53683/callback"
    - "http://localhost:53684/callback"
```

### Full Configuration Example

```yaml
# OAuth Configuration (REQUIRED)
auth:
  client_id: "your-zoom-client-id"
  redirect_url: "http://localhost:53682/callback"
  redirect_urls:                 # Optional callback fallback list
    - "http://localhost:53682/callback"
    - "http://localhost:53683/callback"
    - "http://localhost:53684/callback"

# Default Settings
defaults:
  output_format: table          # table, json, csv
  date_range: 30d               # Default --since value for ls
  download_path: ~/zoom-downloads

# Download Command Configuration
download:
  default_resources:            # Resources included by default
    - summary.md
    - transcript.vtt
    - metadata.json
  exclude_resources:            # Resources excluded by default
    - recording.mp4
    - recording.m4a
  preserve_structure: false     # Use date-prefix by default

# Resource Settings
resources:
  summary_format: md            # md (txt, json not yet supported)
  transcript_format: vtt        # vtt or srt
  include_chat: false           # Include chat logs
  include_metadata: true        # Include metadata.json

# Display Preferences
display:
  use_color: true               # Colored output
  date_format: "2006-01-02"     # Go time format string
  time_format: "15:04"          # Go time format string
  pager: less                   # Pager for long output

# Advanced Settings
advanced:
  api_base_url: "https://api.zoom.us/v2"
  timeout: 30s                  # HTTP timeout
  retry_attempts: 3             # Failed request retries
  cache_ttl: 5m                 # Cache meeting list for 5 minutes
```

### Configuration Schema

#### `auth` (required)

| Key | Type | Description | Required |
|-----|------|-------------|----------|
| `client_id` | string | Zoom OAuth Client ID | Yes |
| `client_secret` | string | Zoom OAuth Client Secret | No (only if app config requires confidential token exchange) |
| `redirect_url` | string | Primary OAuth callback URL | Yes |
| `redirect_urls` | array | Ordered fallback callback URLs | No |

#### `defaults` (optional)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `output_format` | string | `table` | Default output format (table, json, csv) |
| `date_range` | string | `30d` | Default date range for `ls` |
| `download_path` | string | `~/zoom-downloads` | Default download directory |

#### `download` (optional)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `default_resources` | array | `[summary.md, transcript.vtt, metadata.json]` | Resources included in download |
| `exclude_resources` | array | `[recording.mp4, recording.m4a]` | Resources excluded from download |
| `preserve_structure` | bool | `false` | Preserve directory structure |

#### `resources` (optional)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `summary_format` | string | `md` | Summary format (md only for now) |
| `transcript_format` | string | `vtt` | Transcript format (vtt or srt) |
| `include_chat` | bool | `false` | Include chat logs |
| `include_metadata` | bool | `true` | Include metadata.json |

#### `display` (optional)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `use_color` | bool | `true` | Enable colored output |
| `date_format` | string | `2006-01-02` | Date format (Go time format) |
| `time_format` | string | `15:04` | Time format (Go time format) |
| `pager` | string | `less` | Pager for long output |

---

## Output Formats

### Table Format (Default for TTY)

Human-readable tabular output with column alignment.

**Example** (`zoom-cli ls /`):
```
NAME                INSTANCES  LAST MEETING
Team-Standup        15         2026-02-23 10:30
Client-Review       8          2026-02-20 14:00
1on1-with-Manager   12         2026-02-19 09:00
```

**Features**:
- Auto-sized columns
- Color highlighting (if `display.use_color: true`)
- Truncates long values to fit terminal width

### JSON Format (Default when Piped)

Structured JSON output for scripting.

**Example** (`zoom-cli ls / --output json`):
```json
[
  {
    "name": "Team-Standup",
    "normalized_name": "Team-Standup",
    "instance_count": 15,
    "last_meeting": "2026-02-23T10:30:00Z"
  },
  {
    "name": "Client Review",
    "normalized_name": "Client-Review",
    "instance_count": 8,
    "last_meeting": "2026-02-20T14:00:00Z"
  }
]
```

**Features**:
- Pretty-printed with 2-space indentation
- ISO-8601 timestamps
- Machine-parseable

### CSV Format

Comma-separated values for spreadsheet import.

**Example** (`zoom-cli ls / --output csv`):
```csv
name,normalized_name,instance_count,last_meeting
Team Standup,Team-Standup,15,2026-02-23T10:30:00Z
Client Review,Client-Review,8,2026-02-20T14:00:00Z
1:1 with Manager,1on1-with-Manager,12,2026-02-19T09:00:00Z
```

**Features**:
- RFC 4180 compliant
- Header row included
- Quoted values containing commas

### Auto-Detection

The CLI automatically detects output context:
- **TTY** (interactive terminal): Table format
- **Pipe** (scripting): JSON format
- **Override**: Use `--output <format>` flag

---

## Error Handling

### Error Message Format

```
Error: <short description>

<detailed explanation>

<suggested action or command>
```

### Common Error Scenarios

#### Not Authenticated

```bash
$ zoom-cli ls /
Error: Not authenticated

You need to log in first. Run:
  zoom-cli auth login

This will open your browser to complete OAuth authentication.
```

#### Meeting Not Found

```bash
$ zoom-cli ls "/Nonexistent Meeting/"
Error: Meeting series not found: "Nonexistent Meeting"

Did you mean one of these?
  - Team Standup
  - Team Planning
  - Team Retro

Try: zoom-cli ls / to see all available meetings.
```

#### Ambiguous Match

```bash
$ zoom-cli ls /team/
Error: Ambiguous path "/team/" matches multiple series

Matches found:
  - Team Standup
  - Team Planning
  - Team Retro

Use a more specific path or exact name:
  zoom-cli ls "/Team Standup/"
```

#### Resource Not Available

```bash
$ zoom-cli cat "/Team-Standup/@latest/recording.mp4"
Error: Resource not available: recording.mp4

This meeting instance does not have a recording. Available resources:
  - summary.md
  - transcript.vtt
  - metadata.json

Try: zoom-cli ls "/Team-Standup/@latest/" to see all available resources.
```

#### API Error

```bash
$ zoom-cli ls /
Error: API request failed

Status: 429 Too Many Requests
Message: Rate limit exceeded. Please try again in 60 seconds.

The Zoom API has rate limits. Wait a moment and try again.
```

#### Network Error

```bash
$ zoom-cli ls /
Error: Network request failed

Unable to connect to api.zoom.us. Please check:
  - Internet connection
  - Firewall settings
  - VPN configuration

Try: zoom-cli auth status to verify credentials.
```

---

## Example Workflows

### Daily Standup Summary Review

View today's standup summary:
```bash
zoom-cli cat "/Team-Standup/@latest/summary.md"
```

View yesterday's standup:
```bash
zoom-cli ls "/Team-Standup/" --since yesterday
zoom-cli cat "/Team-Standup/@2/summary.md"
```

### Weekly Archive Script

Download all summaries from the last week:
```bash
#!/bin/bash
# weekly-archive.sh

# Create archive directory
ARCHIVE_DIR=~/zoom-archive/$(date +%Y-week-%U)
mkdir -p "$ARCHIVE_DIR"

# Download all summaries from last 7 days
zoom-cli download "/Team-Standup/*/" "$ARCHIVE_DIR/standups/" --since 7d
zoom-cli download "/Client-Review/*/" "$ARCHIVE_DIR/client/" --since 7d
zoom-cli download "/1on1-with-Manager/*/" "$ARCHIVE_DIR/1on1/" --since 7d

echo "Weekly archive complete: $ARCHIVE_DIR"
```

### Search for Action Items

Download all recent summaries and search locally:
```bash
#!/bin/bash
# find-action-items.sh

# Download all summaries from last month
TEMP_DIR=$(mktemp -d)
zoom-cli download "/*/*/summary.md" "$TEMP_DIR" --since 1m

# Search for action items
echo "Action items from last month:"
grep -r -i "action item" "$TEMP_DIR" | \
  sed 's|'"$TEMP_DIR/"'||' | \
  sort

# Cleanup
rm -rf "$TEMP_DIR"
```

### Bulk Download by Meeting Series

Download everything for a specific meeting series:
```bash
#!/bin/bash
# archive-client-reviews.sh

DEST=~/client-reviews-archive

# Download all instances with full structure
zoom-cli download "/Client-Review/*/" "$DEST" --preserve-structure

# Include recordings for important meetings
zoom-cli download "/Client-Review/*/" "$DEST" --preserve-structure --include recording.mp4

echo "Client reviews archived to: $DEST"
```

### Quick Meeting Summary

Create an alias in your shell:
```bash
# Add to ~/.bashrc or ~/.zshrc
alias standup='zoom-cli cat "/Team-Standup/@latest/summary.md"'
alias standup-yesterday='zoom-cli cat "/Team-Standup/@2/summary.md"'
```

Usage:
```bash
standup           # View today's standup summary
standup-yesterday # View yesterday's summary
```

### Export to CSV for Analysis

Export all meetings from a date range:
```bash
zoom-cli ls "/Team-Standup/" --since 2026-01-01 --output csv > standups-jan.csv
```

Open in spreadsheet or analyze with tools like `csvkit`:
```bash
csvstat standups-jan.csv
csvcut -c date,duration,participants standups-jan.csv | csvlook
```

### Download Latest Recordings

Download only the latest recording from each series:
```bash
#!/bin/bash
# download-latest-recordings.sh

for series in "Team-Standup" "All-Hands" "Client-Review"; do
  zoom-cli cp "/$series/@latest/recording.mp4" ~/recordings/
done
```

### Automated Daily Sync (Cron Job)

Add to crontab (`crontab -e`):
```bash
# Download new summaries every day at 6 PM
0 18 * * * /usr/local/bin/zoom-cli download "/Team-Standup/*/" ~/zoom-archive/standups/ --since 1d
```

Or use a systemd timer (Linux):
```ini
# /etc/systemd/user/zoom-sync.timer
[Unit]
Description=Daily Zoom summary sync

[Timer]
OnCalendar=daily
OnCalendar=18:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/user/zoom-sync.service
[Unit]
Description=Sync Zoom summaries

[Service]
Type=oneshot
ExecStart=/usr/local/bin/zoom-cli download "/Team-Standup/*/" %h/zoom-archive/standups/ --since 1d
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-23 | Initial design specification |

---

**End of CLI Design**
