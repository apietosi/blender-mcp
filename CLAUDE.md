# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BlenderMCP connects Blender 3D to Claude AI via the Model Context Protocol (MCP). It has two main components:
1. **Blender addon** (`addon.py`) — runs a socket server inside Blender on port 9876
2. **MCP server** (`src/blender_mcp/server.py`) — Claude-facing MCP tool implementations

## Development Commands

Uses [UV](https://docs.astral.sh/uv/) as the package manager. Python 3.10+ required (`.python-version` pins 3.13.2).

```bash
uv sync                # Install dependencies
uv run blender-mcp     # Run the MCP server directly
uvx blender-mcp        # Run via uvx (used in Claude/Cursor integrations)
```

Environment variables:
```bash
BLENDER_HOST=localhost  # default
BLENDER_PORT=9876       # default
DISABLE_TELEMETRY=true  # opt-out of telemetry
```

There are no test commands configured in the project.

## Architecture

### Communication Flow
Claude → MCP server (`server.py`) → TCP socket → Blender addon (`addon.py`) → Blender Python API

`BlenderConnection` in `server.py` manages the TCP socket connection as a singleton. Commands are sent as JSON and responses are received in chunks (180s timeout). The addon runs a persistent socket server thread inside Blender.

### Key Files
- `addon.py` — Blender extension (111KB): socket server, all actual Blender operations, asset downloads, AI model generation API calls
- `src/blender_mcp/server.py` — MCP server (1,185 lines): 22 tool definitions using FastMCP, `BlenderConnection` class, server lifecycle management
- `src/blender_mcp/telemetry.py` — Anonymous telemetry with two-tier consent model (with/without user consent)
- `src/blender_mcp/telemetry_decorator.py` — Decorator for automatic telemetry on tool calls
- `src/blender_mcp/config.py` — **Gitignored secrets file** (must be created locally); contains `telemetry_config` with Supabase URL/key

### External Integrations
The addon and server both integrate with:
- **PolyHaven** — HDRI, textures, 3D models (REST API)
- **Sketchfab** — 3D model search and download
- **Hyper3D/Rodin** — AI text/image-to-3D generation
- **Hunyuan3D** — Alternative AI 3D generation

### Adding New MCP Tools
1. Implement the Blender-side operation in `addon.py` (receives JSON command, returns JSON result)
2. Add a `@mcp.tool()` decorated function in `server.py` that calls `blender.send_command()`
3. Wrap with `@track_tool_execution` from `telemetry_decorator.py` for telemetry

### Telemetry
The telemetry system stores to Supabase. Without user consent: only tool name, success/failure, duration. With consent: prompts, code, screenshots. Customer UUID persists at `~/.local/share/BlenderMCP/customer_uuid.txt`. Disable via any of: `DISABLE_TELEMETRY`, `BLENDER_MCP_DISABLE_TELEMETRY`, or `MCP_DISABLE_TELEMETRY` env vars.
