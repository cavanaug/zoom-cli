# zoom-cli

A command-line interface for managing Zoom meetings using a filesystem metaphor.

> **Status**: 🚧 Under Development - Not yet functional

## Overview

`zoom-cli` treats your Zoom meetings as a virtual filesystem where you can browse, view, and download meeting summaries, transcripts, and recordings using familiar Unix commands like `ls`, `cat`, and `cp`.

```bash
# List all your meeting series
zoom-cli ls /

# View the latest standup summary
zoom-cli cat "/Team-Standup/@latest/summary.md"

# Download summaries from the last week
zoom-cli download "/Team-Standup/*/" ~/summaries/ --since 7d
```

## Key Features

- **Filesystem metaphor** - Navigate meetings like directories
- **AI summaries** - Access Zoom's AI-generated meeting summaries
- **Transcripts** - Download meeting transcripts in VTT/SRT format
- **Recordings** - Download video/audio recordings
- **Smart filtering** - Filter by date, resources, and more
- **Multiple output formats** - Table, JSON, or CSV output
- **Scriptable** - Designed for both interactive and automated use

## Documentation

- **[CLI-DESIGN.md](./CLI-DESIGN.md)** - Command usage, syntax, and workflows
- **[API-REFERENCE.md](./API-REFERENCE.md)** - Zoom API integration details
- **[IMPLEMENTATION.md](./IMPLEMENTATION.md)** - Development guide and architecture

## Quick Start

### Prerequisites

- Go 1.21 or later
- A Zoom account with meetings/recordings
- Zoom OAuth app credentials (see [setup guide](./IMPLEMENTATION.md#oauth-app-setup))

### Installation

```bash
# Clone the repository
git clone https://github.com/cavanaug/zoom-cli.git
cd zoom-cli

# Build
go build -o zoom-cli main.go

# Run
./zoom-cli --help
```

### Configuration

Create `~/.config/zoom-cli/config.yaml`:

```yaml
auth:
  client_id: "your-zoom-client-id"
  client_secret: "your-zoom-client-secret"
  redirect_url: "http://localhost:8080/callback"
```

### Authentication

```bash
# Authenticate with Zoom
zoom-cli auth login

# Check authentication status
zoom-cli auth status
```

## Example Usage

```bash
# List all meeting series
zoom-cli ls /

# List Team Standup meetings
zoom-cli ls "/Team-Standup/"

# View latest standup summary
zoom-cli cat "/Team-Standup/@latest/summary.md"

# Download summaries from last 7 days
zoom-cli download "/Team-Standup/*/" ~/Downloads/ --since 7d
```

## Development Status

This project is currently in early development. See [IMPLEMENTATION.md](./IMPLEMENTATION.md) for the development roadmap.

### Implementation Phases

- [ ] **Phase 0**: Foundation & Authentication
- [ ] **Phase 1**: Core Filesystem Metaphor (Read-Only)
- [ ] **Phase 2**: Downloads (Summaries & Transcripts)
- [ ] **Phase 3**: Sync & State Tracking (Deferred)
- [ ] **Phase 4**: Recordings & Advanced Features
- [ ] **Phase 5**: Polish & Production Ready

## Contributing

This is a personal project currently under active development. Contributions, issues, and feature requests are welcome!

## License

MIT License - See [LICENSE](./LICENSE) for details

## Acknowledgments

- Inspired by the Unix philosophy of treating everything as a file
- Built with [Cobra](https://github.com/spf13/cobra) for CLI framework
- Uses Zoom's official API for meeting data
