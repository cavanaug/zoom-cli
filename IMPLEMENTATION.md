# zoom-cli Implementation Specification

**Version**: 1.0  
**Last Updated**: 2026-02-23

## Overview

This document focuses on **implementation design decisions** for `zoom-cli`, including technology stack, project structure, development phases, and build instructions.

For user-facing documentation, see:
- **[CLI-DESIGN.md](./CLI-DESIGN.md)** - Command usage, syntax, configuration, and workflows
- **[API-REFERENCE.md](./API-REFERENCE.md)** - Zoom API integration details and OAuth implementation

---

## Table of Contents

1. [Technology Stack](#technology-stack)
2. [Project Structure](#project-structure)
3. [Implementation Phases](#implementation-phases)
4. [Development Setup](#development-setup)
5. [Testing Strategy](#testing-strategy)
6. [Build and Release](#build-and-release)

---

## Technology Stack

### Language

**Go** (Golang) - chosen for:
- Cross-platform compilation (Linux, macOS, Windows)
- Single binary distribution (no runtime dependencies)
- Excellent standard library for HTTP, OAuth, file I/O
- Strong CLI tooling ecosystem
- Fast compilation and execution

### Core Dependencies

| Library | Purpose | Rationale |
|---------|---------|-----------|
| `github.com/spf13/cobra` | CLI framework | Industry standard, excellent subcommand support, auto-generated help |
| `golang.org/x/oauth2` | OAuth 2.0 implementation | Official Go OAuth library, handles token refresh automatically |
| `gopkg.in/yaml.v3` | YAML parsing | Configuration file parsing, well-maintained |
| `github.com/fatih/color` | Colored terminal output | Cross-platform color support, simple API |
| `github.com/schollz/progressbar/v3` | Progress bars | Download progress indication |
| `github.com/olekukonko/tablewriter` | Table formatting | Pretty table output for `ls` command |

### Standard Library Usage

- `net/http` - HTTP client for Zoom API calls
- `encoding/json` - JSON parsing for API responses
- `path/filepath` - Filesystem path handling
- `time` - Date/time parsing and manipulation
- `os` - File I/O and environment variables

## Project Structure

### Directory Layout

```
zoom-cli/
├── cmd/
│   ├── root.go           # Root command setup (cobra)
│   ├── auth.go           # Auth subcommands (login, logout, status, whoami)
│   ├── config.go         # Config subcommands (list, get, set, edit, help, init)
│   ├── ls.go             # List command
│   ├── cat.go            # Cat command
│   ├── cp.go             # Copy command
│   └── download.go       # Download command
├── internal/
│   ├── auth/
│   │   ├── oauth.go      # OAuth 2.0 flow implementation
│   │   └── token.go      # Token storage and refresh
│   ├── config/
│   │   ├── config.go     # Config file management
│   │   └── schema.go     # Config schema and validation
│   ├── api/
│   │   ├── client.go     # Zoom API HTTP client
│   │   ├── endpoints.go  # API endpoint definitions
│   │   ├── meetings.go   # Meeting-related API calls
│   │   ├── summaries.go  # Summary-related API calls
│   │   └── recordings.go # Recording-related API calls
│   ├── vfs/
│   │   ├── path.go       # Path parsing and normalization
│   │   ├── resolver.go   # Path resolution (fuzzy matching, globs)
│   │   ├── aliases.go    # Special alias handling (@latest, @1, etc.)
│   │   └── filesystem.go # Virtual filesystem abstraction
│   ├── datetime/
│   │   ├── parser.go     # Date/time parsing (Taskwarrior-style)
│   │   └── relative.go   # Relative date calculation
│   ├── output/
│   │   ├── table.go      # Table formatter
│   │   ├── json.go       # JSON formatter
│   │   ├── csv.go        # CSV formatter
│   │   └── format.go     # Format detection and routing
│   ├── download/
│   │   ├── downloader.go # Download orchestration
│   │   ├── progress.go   # Progress tracking
│   │   └── organize.go   # File organization logic
│   └── errors/
│       ├── errors.go     # Custom error types
│       └── messages.go   # User-friendly error messages
├── main.go               # Entry point
├── go.mod                # Go module definition
├── go.sum                # Dependency checksums
├── SPECIFICATION.md      # This file (implementation spec)
├── CLI-DESIGN.md         # User-facing command documentation
├── API-REFERENCE.md      # Zoom API integration details
├── README.md             # Getting started guide
├── LICENSE               # License file
└── .gitignore            # Git ignore rules
```

### Module Organization

- **`cmd/`** - Command definitions using Cobra
  - Each command in its own file
  - Command logic delegates to `internal/` packages
  
- **`internal/auth/`** - OAuth 2.0 authentication
  - Browser-based OAuth flow
  - Token storage and automatic refresh
  
- **`internal/api/`** - Zoom API client
  - HTTP client with retry logic and rate limiting
  - Endpoint-specific methods
  
- **`internal/vfs/`** - Virtual filesystem abstraction
  - Path parsing and resolution
  - Fuzzy matching and glob support
  
- **`internal/datetime/`** - Date/time parsing
  - Taskwarrior-style syntax support
  - Relative date calculation
  
- **`internal/output/`** - Output formatters
  - Table, JSON, CSV formats
  - TTY vs pipe detection
  
- **`internal/download/`** - File download logic
  - Streaming downloads with progress
  - File organization strategies
  
- **`internal/errors/`** - Error handling
  - Custom error types
  - User-friendly error messages

---

## Implementation Phases

### Phase 0: Foundation & Authentication

**Goal**: OAuth login working, tokens stored securely

**Key Implementation Tasks**:
- [ ] Initialize Go project
  ```bash
  go mod init github.com/yourusername/zoom-cli
  ```
- [ ] Install core dependencies
  ```bash
  go get github.com/spf13/cobra
  go get golang.org/x/oauth2
  go get gopkg.in/yaml.v3
  ```
- [ ] Implement OAuth 2.0 authorization code flow (`internal/auth/oauth.go`)
  - Browser launch for authorization
  - Local HTTP callback server (`localhost:8080`)
  - Token exchange
- [ ] Implement token storage (`internal/auth/token.go`)
  - Secure storage: `~/.config/zoom-cli/tokens.json` with `0600` permissions
  - Auto-refresh mechanism (refresh 5 min before expiration)
- [ ] Implement config file support (`internal/config/`)
  - YAML parser
  - Config file location: `~/.config/zoom-cli/config.yaml`
  - Default config values
- [ ] Implement `auth` commands (`cmd/auth.go`)
  - `zoom-cli auth login`
  - `zoom-cli auth logout`
  - `zoom-cli auth status`
  - `zoom-cli auth whoami`
- [ ] Implement `config` commands (`cmd/config.go`)
  - `zoom-cli config list`
  - `zoom-cli config get <key>`
  - `zoom-cli config set <key> <value>`
  - `zoom-cli config edit`
  - `zoom-cli config init`

**Success Criteria**:
- ✓ Can authenticate via browser OAuth flow
- ✓ Tokens stored securely and auto-refresh
- ✓ Config file created and managed
- ✓ `auth status` and `whoami` display user info

**Estimated Effort**: 3-5 days

---

### Phase 1: Core Filesystem Metaphor (Read-Only)

**Goal**: Browse meetings and view summaries

**Key Implementation Tasks**:
- [ ] Implement Zoom API client (`internal/api/client.go`)
  - HTTP client with auto-refresh auth
  - Error handling and retries (exponential backoff)
  - Rate limit handling (check `X-RateLimit-Remaining` header)
  - Endpoints: `/users/me`, `/users/me/meetings`, `/meetings/{id}`, `/meetings/{id}/meeting_summary`
- [ ] Implement virtual filesystem abstraction (`internal/vfs/`)
  - Path parsing (quoted, escaped, glob)
  - Path resolution with fuzzy matching
  - Meeting series normalization (spaces → hyphens, case preserved)
  - Special alias support (`@latest`, `@1`, `@2`, etc.)
- [ ] Implement date/time parsing (`internal/datetime/`)
  - Taskwarrior-style date syntax
  - Absolute dates (ISO-8601)
  - Relative dates (`7d`, `yesterday`, `sow`, etc.)
- [ ] Implement `ls` command (`cmd/ls.go`)
  - List meeting series: `zoom-cli ls /`
  - List instances: `zoom-cli ls "/Team-Standup/"`
  - List resources: `zoom-cli ls "/Team-Standup/@latest/"`
  - Glob pattern support
  - `--since` flag for date filtering
  - `-l` flag for detailed view
- [ ] Implement output formatters (`internal/output/`)
  - Table format (default for TTY) using `olekukonko/tablewriter`
  - JSON format (default for pipe)
  - CSV format
  - Auto-detection (TTY vs pipe using `os.Stdout.Stat()`)
  - Color support using `fatih/color` (configurable)
- [ ] Implement `cat` command (`cmd/cat.go`)
  - View summaries: `zoom-cli cat "/Team-Standup/@latest/summary.md"`
  - View metadata: `zoom-cli cat "/Meeting/@latest/metadata.json"`
  - Pretty-printing (JSON, Markdown)
  - Pager integration for long output

**API Endpoints Used** (see [API-REFERENCE.md](./API-REFERENCE.md) for details):
- `GET /users/me` - User profile
- `GET /users/me/meetings` - List meetings
- `GET /meetings/{meetingId}` - Meeting details
- `GET /meetings/{meetingId}/meeting_summary` - AI summary

**Success Criteria**:
- ✓ Can list all meeting series
- ✓ Can list instances for a specific series
- ✓ Can view summary for any meeting
- ✓ Fuzzy matching works for meeting names
- ✓ `@latest` and `@N` aliases work
- ✓ Date filtering works (`--since 7d`)
- ✓ Output formats (table, JSON, CSV) work correctly

**Estimated Effort**: 5-7 days

---

### Phase 2: Downloads (Summaries & Transcripts)

**Goal**: Download summaries and transcripts to local filesystem

**Key Implementation Tasks**:
- [ ] Extend Zoom API client for file downloads (`internal/api/recordings.go`)
  - `GET /meetings/{meetingId}/recordings` - List recordings/transcripts
  - Download transcript files (VTT/SRT) with Bearer token authentication
- [ ] Implement `cp` command (`cmd/cp.go`)
  - Standard `cp` semantics
  - Single file copy: `zoom-cli cp "/Path/file" ~/dest/`
  - Recursive copy: `zoom-cli cp -r "/Path/" ~/dest/`
  - Glob pattern support
  - Preserve exact directory structure
- [ ] Implement `download` command (`cmd/download.go`)
  - Config-driven resource filtering
  - Date-prefix flat structure by default
  - `--preserve-structure` flag for hierarchy
  - `--since <date>` for date filtering
  - `--include` and `--exclude` flags
  - `--dry-run` flag
- [ ] Implement download management (`internal/download/`)
  - Streaming downloads for large files
  - Progress indication using `schollz/progressbar`
  - File organization logic (date-prefix vs structure-preserving)
  - Conflict resolution (overwrite, skip, rename)
- [ ] Extend virtual filesystem for transcripts
  - `transcript.vtt` resource
  - `transcript.srt` resource (if available)
  - Format selection via config

**API Endpoints Used** (see [API-REFERENCE.md](./API-REFERENCE.md) for details):
- `GET /meetings/{meetingId}/recordings` - List recording files
- `GET {download_url}` - Download files with Bearer token

**Success Criteria**:
- ✓ Can copy single files to local filesystem
- ✓ Can copy entire instances recursively
- ✓ `download` respects config defaults (excludes recordings)
- ✓ `download` can override config with flags
- ✓ Date-prefix organization works correctly
- ✓ `--preserve-structure` creates proper hierarchy
- ✓ Transcripts download correctly (VTT format)
- ✓ Progress indication for downloads

**Estimated Effort**: 5-7 days

---

### Phase 3: Sync & State Tracking (Deferred)

**Status**: Design deferred pending further discussion

**Goal**: Intelligent syncing with "since last run" support

**Placeholder Tasks**:
- [ ] Design state tracking mechanism
- [ ] Implement `sync` command
- [ ] State file management
- [ ] Incremental sync logic

**Notes**: This phase is on hold while the sync mechanism is being reconsidered.

---

### Phase 4: Recordings & Advanced Features

**Goal**: Full resource support and advanced filtering

**Key Implementation Tasks**:
- [ ] Extend Zoom API client for recordings
  - Download video files (MP4)
  - Download audio files (M4A)
  - Handle large file downloads efficiently (streaming, resume on failure)
- [ ] Extend Zoom API client for chat logs
  - `GET /meetings/{meetingId}/chat` (if available)
  - Format chat messages nicely
- [ ] Add recording support to virtual filesystem
  - `recording.mp4` resource
  - `recording.m4a` resource
  - Size indicators in `ls` output
- [ ] Add chat log support
  - `chat.txt` resource
  - Pretty formatting
- [ ] Performance optimizations
  - API response caching (5-15 min TTL)
  - Parallel API requests
  - Connection pooling

**Success Criteria**:
- ✓ Can download video recordings
- ✓ Can download audio-only recordings
- ✓ Chat logs download and display correctly
- ✓ Large file downloads are efficient

**Estimated Effort**: 4-6 days

---

### Phase 5: Polish & Production Ready

**Goal**: Production-ready CLI with excellent UX

**Key Implementation Tasks**:
- [ ] Shell completion
  - Bash completion script
  - Zsh completion script
  - Fish completion script
- [ ] Enhanced error messages
  - Contextual suggestions
  - Recovery actions
  - Levenshtein distance for "did you mean" suggestions
- [ ] Logging framework
  - Debug mode (`--debug` flag)
  - Structured logging to file
- [ ] Testing
  - Unit tests for core functions
  - Integration tests with mock API
  - End-to-end tests
  - Target: > 80% test coverage
- [ ] Documentation
  - Comprehensive README
  - Installation guide
  - Configuration guide
  - Example scripts
  - Man pages
  - Changelog
- [ ] Build & Release
  - Cross-platform builds (Linux, macOS, Windows)
  - GitHub Actions CI/CD
  - Release automation
  - Homebrew formula
  - Debian/RPM packages

**Success Criteria**:
- ✓ Shell completion works in bash/zsh/fish
- ✓ Error messages are clear and actionable
- ✓ Debug logging available
- ✓ Test coverage > 80%
- ✓ Complete documentation
- ✓ Multi-platform binaries available
- ✓ Installation is simple

**Estimated Effort**: 5-7 days

---

## Development Setup

### Prerequisites

- **Go 1.21+** - [Install Go](https://golang.org/doc/install)
- **Git** - For version control
- **Zoom OAuth App** - See [OAuth App Setup](#oauth-app-setup) below

### Initial Setup

```bash
# Clone repository
git clone https://github.com/yourusername/zoom-cli.git
cd zoom-cli

# Initialize Go module
go mod init github.com/yourusername/zoom-cli

# Install dependencies
go get github.com/spf13/cobra
go get golang.org/x/oauth2
go get gopkg.in/yaml.v3
go get github.com/fatih/color
go get github.com/schollz/progressbar/v3
go get github.com/olekukonko/tablewriter

# Tidy dependencies
go mod tidy

# Build
go build -o zoom-cli main.go

# Run
./zoom-cli --help
```

### OAuth App Setup

1. Go to [Zoom Marketplace](https://marketplace.zoom.us/)
2. Click **Develop** → **Build App**
3. Select **User-managed app** (OAuth)
4. Fill in basic information:
   - App name: `zoom-cli`
   - Short description: CLI tool for managing Zoom meetings
   - Company name: Your name
5. Set **Redirect URL for OAuth**: `http://localhost:8080/callback`
6. Add **Scopes**:
   - `meeting:read` - List and view meetings
   - `meeting_summary:read` - Access AI summaries
   - `recording:read` - List and download recordings
   - `user:read` - Get user profile information
7. Copy **Client ID** and **Client Secret**
8. Create config file (`~/.config/zoom-cli/config.yaml`):
   ```yaml
   auth:
     client_id: "your-client-id-here"
     client_secret: "your-client-secret-here"
     redirect_url: "http://localhost:8080/callback"
   ```

### Development Workflow

```bash
# Run during development
go run main.go <command>

# Build for local testing
go build -o zoom-cli main.go

# Install to GOPATH/bin
go install

# Run tests (once implemented)
go test ./...

# Run tests with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Format code
go fmt ./...

# Lint code (requires golangci-lint)
golangci-lint run
```

---

## Testing Strategy

### Unit Tests

**Location**: `*_test.go` files alongside source files

**Coverage Target**: > 80%

**Key Areas to Test**:
- Path parsing and normalization (`internal/vfs/path_test.go`)
- Date/time parsing (`internal/datetime/parser_test.go`)
- Config file parsing (`internal/config/config_test.go`)
- Token refresh logic (`internal/auth/token_test.go`)
- Output formatters (`internal/output/*_test.go`)

**Example Test Structure**:
```go
package vfs

import "testing"

func TestNormalizeName(t *testing.T) {
    tests := []struct {
        input    string
        expected string
    }{
        {"Team Standup", "Team-Standup"},
        {"1:1 Meeting", "1on1-Meeting"},
        {"Team  Standup", "Team-Standup"},
    }
    
    for _, tt := range tests {
        t.Run(tt.input, func(t *testing.T) {
            result := NormalizeName(tt.input)
            if result != tt.expected {
                t.Errorf("got %q, want %q", result, tt.expected)
            }
        })
    }
}
```

### Integration Tests

**Location**: `tests/integration/`

**Approach**: Mock Zoom API with `httptest` server

**Key Scenarios**:
- OAuth flow (mock authorization and token exchange)
- API client (mock API responses)
- End-to-end command execution

**Example**:
```go
package integration

import (
    "net/http/httptest"
    "testing"
)

func TestListMeetings(t *testing.T) {
    // Create mock API server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Mock /users/me/meetings response
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"meetings": [{"topic": "Test Meeting"}]}`))
    }))
    defer server.Close()
    
    // Test ls command against mock API
    // ...
}
```

### End-to-End Tests

**Location**: `tests/e2e/`

**Approach**: Test against real Zoom API with test account

**Prerequisites**:
- Test Zoom account
- OAuth app credentials in environment variables

**Run Manually**: Due to OAuth requirements, E2E tests may need manual triggering

---

## Build and Release

### Local Build

```bash
# Development build
go build -o zoom-cli main.go

# Build with version info
VERSION=1.0.0
go build -ldflags "-X main.Version=$VERSION" -o zoom-cli main.go
```

### Cross-Platform Builds

```bash
# Linux AMD64
GOOS=linux GOARCH=amd64 go build -o zoom-cli-linux-amd64 main.go

# macOS AMD64 (Intel)
GOOS=darwin GOARCH=amd64 go build -o zoom-cli-darwin-amd64 main.go

# macOS ARM64 (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -o zoom-cli-darwin-arm64 main.go

# Windows AMD64
GOOS=windows GOARCH=amd64 go build -o zoom-cli-windows-amd64.exe main.go
```

### Release Automation (GitHub Actions)

**`.github/workflows/release.yml`**:
```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      
      - name: Build binaries
        run: |
          GOOS=linux GOARCH=amd64 go build -o zoom-cli-linux-amd64
          GOOS=darwin GOARCH=amd64 go build -o zoom-cli-darwin-amd64
          GOOS=darwin GOARCH=arm64 go build -o zoom-cli-darwin-arm64
          GOOS=windows GOARCH=amd64 go build -o zoom-cli-windows-amd64.exe
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            zoom-cli-linux-amd64
            zoom-cli-darwin-amd64
            zoom-cli-darwin-arm64
            zoom-cli-windows-amd64.exe
```

### Package Distribution

#### Homebrew Formula

**`homebrew-tap/zoom-cli.rb`**:
```ruby
class ZoomCli < Formula
  desc "CLI tool for managing Zoom meetings"
  homepage "https://github.com/yourusername/zoom-cli"
  url "https://github.com/yourusername/zoom-cli/archive/v1.0.0.tar.gz"
  sha256 "..."
  license "MIT"

  depends_on "go" => :build

  def install
    system "go", "build", *std_go_args
  end

  test do
    system "#{bin}/zoom-cli", "--version"
  end
end
```

Installation:
```bash
brew tap yourusername/tap
brew install zoom-cli
```

#### Debian Package

Use `nfpm` or `fpm` to create `.deb` packages for Debian/Ubuntu.

#### RPM Package

Use `nfpm` or `fpm` to create `.rpm` packages for RedHat/CentOS/Fedora.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-23 | Initial implementation specification |

---

**End of Implementation Specification**

For user-facing documentation, see:
- **[CLI-DESIGN.md](./CLI-DESIGN.md)** - Command usage, workflows, and examples
- **[API-REFERENCE.md](./API-REFERENCE.md)** - Zoom API details and OAuth implementation
