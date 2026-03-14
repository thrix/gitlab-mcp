# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is `@zereight/mcp-gitlab`, a GitLab MCP (Model Context Protocol) server that exposes ~98 GitLab API operations as MCP tools. It supports stdio, SSE, and Streamable HTTP transports, with authentication via personal access tokens, OAuth2, or remote per-session authorization.

## Build and Development Commands

```bash
npm install          # Install dependencies
npm run build        # Compile TypeScript to build/
npm run dev          # Build and run the server
npm run watch        # TypeScript watch mode
npm run lint         # ESLint
npm run lint:fix     # ESLint with auto-fix
npm run format       # Prettier format
npm run format:check # Prettier check
```

## Testing

```bash
npm test                    # Run all tests (build + mock + live)
npm run test:mock           # Run mock tests only (no GitLab credentials needed)
npm run test:live           # Run live API validation (needs .env config)
npm run test:remote-auth    # Remote authorization tests (mock GitLab server)
npm run test:oauth          # OAuth flow tests
npm run test:list-merge-requests  # MR listing tests
npm run test:approvals      # MR approval tests
```

Tests are in `test/` and use Node's built-in test runner (`node --test`) and `tsx` for TypeScript execution. No test framework like Jest or Mocha.

## Architecture

### Source Files (all in root directory)

- **`index.ts`** (~7900 lines) — The monolithic main file containing:
  - CLI argument parsing and config resolution (CLI args > env vars > defaults)
  - MCP Server creation with tool filtering pipeline (toolsets → individual tools → legacy flags → read-only → regex deny)
  - All GitLab API wrapper functions (create/get/list/update/delete for issues, MRs, branches, pipelines, wikis, milestones, labels, releases, etc.)
  - Tool definitions array (`allTools`) mapping tool names to Zod schemas and handlers
  - Transport setup (stdio, SSE via Express, Streamable HTTP via Express)
  - Remote authorization with session management
  - Rate limiting and session capacity enforcement
- **`schemas.ts`** (~2700 lines) — Zod schemas for all GitLab API request/response types
- **`customSchemas.ts`** — Small custom Zod helpers (e.g., `flexibleBoolean`)
- **`oauth.ts`** (~640 lines) — OAuth2 + PKCE authentication flow with token persistence
- **`gitlab-client-pool.ts`** (~140 lines) — HTTP/HTTPS agent pool for connection reuse across multiple GitLab instances

### Key Patterns

- **Tool filtering pipeline**: Tools go through a 6-step filter in `createServer()`: toolset membership → `GITLAB_TOOLS` additions → legacy flag overrides (`USE_PIPELINE`, etc.) → read-only mode → `GITLAB_DENIED_TOOLS_REGEX` → Gemini `$schema` cleanup
- **Toolset system**: Tools are grouped into toolsets (defined in `TOOLSET_DEFINITIONS`). Default toolsets: `merge_requests`, `issues`, `repositories`, `branches`, `projects`, `labels`, `releases`, `users`. Optional: `pipelines`, `milestones`, `wiki`
- **Dynamic API URL**: Per-request GitLab API URL via `AsyncLocalStorage`, allowing tools to target different GitLab instances
- **Each transport connection gets its own `Server` instance** to prevent cross-client data leakage

### Configuration

All configuration via environment variables or CLI args (CLI takes precedence). See `.env.example` for full list. Key vars: `GITLAB_PERSONAL_ACCESS_TOKEN`, `GITLAB_API_URL`, `GITLAB_READ_ONLY_MODE`, `GITLAB_TOOLSETS`, `GITLAB_TOOLS`, `GITLAB_DENIED_TOOLS_REGEX`.

## Tech Stack

- TypeScript (ES2022, Node16 module resolution)
- Node.js >= 18 (`.nvmrc`: 22.21.1)
- `@modelcontextprotocol/sdk` for MCP protocol
- `zod` for schema validation, `zod-to-json-schema` for tool schema generation
- `express` for HTTP transports
- `pino` for logging
- Published to npm as `@zereight/mcp-gitlab`

## Release

```bash
npm run release    # Runs scripts/release.sh
npm run deploy     # npm publish --access public
```
