# AGENTS.md

> **Note:** `CLAUDE.md` is a symlink to `AGENTS.md`. Edit `AGENTS.md` to change this content.

## Project Overview

Obsidian plugin that randomly surfaces notes from your vault for review and tracks your progress. Creates a snapshot of all markdown files, then lets you work through them one by one, marking each as reviewed.

## Development Commands

```bash
bun install              # Install dependencies
bun run dev              # Watch mode with auto-rebuild
bun run build            # Production build (runs check first)
bun run check            # Run all checks (typecheck + biome)
bun run typecheck        # TypeScript type checking only
bun run lint             # Biome lint + format check
bun run lint:fix         # Auto-fix lint and format issues
bun run format           # Format code with Biome
bun run validate         # Full validation (types, checks, build, output)
bun run version          # Sync package.json version → manifest.json + versions.json
bun test                 # Run tests
```

## Architecture

### Build System

- **Build script**: `build.ts` uses Bun's native bundler
- **Entry point**: `src/main.ts`
- **Output**: `./main.js` (CommonJS format, minified in production)
- **Externals**: `obsidian` and `electron` are not bundled

### Plugin Structure

- **Snapshot system**: Takes a snapshot of all markdown files, tracks review status per file (to_review, reviewed, deleted)
- **Commands**: Open random unreviewed file, mark as reviewed, review-and-next, unreview
- **Status bar**: Shows review status of the active file
- **Settings tab**: Create/delete snapshots, add new files, toggle status bar

### Version Management

`version-bump.ts` syncs `package.json` version to `manifest.json` and `versions.json`. Run via `bun run version` after updating `package.json`.

### Release Process

Tag and push to trigger the GitHub Actions release workflow:

```bash
git tag -a 1.0.0 -m "Release 1.0.0"
git push origin 1.0.0
```

## Code Style

Enforced by Biome (`biome.json`): 2-space indent, organized imports, git-aware VCS integration.
