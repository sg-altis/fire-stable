# Permission Archetypes

Three Claude Code `.claude/settings.json` templates for different project types. Copy the right one into any new project to get sane defaults without clicking "Yes" 50 times.

## Templates

Stored at `~/.claude/templates/`. Usage:

```bash
mkdir -p .claude && cp ~/.claude/templates/{researcher,builder,restricted}.json .claude/settings.json
```

## The three archetypes

```
                        researcher    builder       restricted
────────────────────    ──────────    ───────────   ──────────
Read/Glob/Grep          ✅ auto       ✅ auto        ✅ auto
Write/Edit              ❌ denied     ✅ auto        ✅ auto
WebFetch/WebSearch      ✅ auto       ✅ auto        ❌ denied
curl                    ✅ auto       ✅ auto        ❌ denied
git read (status/log)   ✅ auto       ✅ auto        ✅ auto
git write (commit/push) ❌ denied     ✅ auto*       ❌ push denied
npm/pip/make            ❌ denied     ✅ auto        ❌ denied
rm/sudo                 ❌ denied     ❌ denied      ❌ denied
.env / credentials      ❌ denied     ❌ denied      ❌ denied

* builder denies --force push and --hard reset
```

### 1. Researcher (`researcher.json`)

**Use for:** wiki repos, research projects, exploration, due diligence, reading codebases you don't own.

**Philosophy:** read everything, fetch anything, write nothing. The agent observes and reports.

**Allows:**
- All file reading (Read, Glob, Grep)
- All web access (WebFetch, WebSearch, curl)
- All git read operations (status, log, diff, show)
- GitHub CLI (gh) for reading issues/PRs
- Python one-liners for data processing
- Version/help flags on any tool

**Denies:**
- All file writing (Write, Edit)
- All git write operations (commit, push, add, checkout, reset)
- All package management (npm, pip)
- All destructive operations (rm, sudo, chmod)

**Example projects:** in3eption, due diligence research, competitor analysis, codebase audits.

### 2. Builder (`builder.json`)

**Use for:** code projects where the agent should build things. The standard development workflow.

**Philosophy:** do anything a developer would, but don't blow up production.

**Allows:**
- All file operations (Read, Write, Edit, Glob, Grep)
- All web access (WebFetch, WebSearch, curl)
- All git operations (commit, push, branch)
- All build tools (make, npm, pip, cargo, go, docker, poetry, pytest)
- File management (mkdir, cp, mv, touch)

**Denies:**
- Force push / hard reset (destructive git)
- sudo (privilege escalation)
- chmod 777 (security footgun)
- Reading .env / credentials files

**Example projects:** butler, fire-stable (when code starts), any tool repo.

### 3. Restricted (`restricted.json`)

**Use for:** sensitive projects, air-gapped work, anything where the agent should stay in its sandbox.

**Philosophy:** read and write files in the folder, nothing else. No network, no git push, no package installs.

**Allows:**
- All file operations (Read, Write, Edit, Glob, Grep)
- Local file management (ls, find, cat, mkdir, cp, mv, touch)

**Denies:**
- All web access (WebFetch, WebSearch, curl, wget)
- All network operations (ssh, scp, rsync)
- git push / remote operations
- npm publish / docker push
- sudo, destructive rm
- Reading .env / credentials files

**Example projects:** confidential documents, client data analysis, sensitive configs, private repos you don't want leaked.

## How Claude Code permissions work

File: `.claude/settings.json` (checked into git, team-wide)
Override: `.claude/settings.local.json` (gitignored, personal)

```json
{
  "permissions": {
    "allow": ["Tool(pattern)", ...],
    "deny": ["Tool(pattern)", ...],
    "ask": ["Tool(pattern)", ...]
  }
}
```

**Evaluation order:** deny → ask → allow (first match wins).

**Bash patterns:** `Bash(npm run *)` — the space before `*` enforces word boundary.

**Precedence:** managed settings > CLI args > local project > shared project > user global.

## Future: Gemini-as-search archetype

TODO: Create a 4th archetype that includes a Gemini Flash MCP server for "Google synthesis" — use Gemini 2.5 Flash as a search + summarization tool instead of raw WebSearch. Would go in the researcher archetype as an MCP tool.

```json
{
  "mcpServers": {
    "gemini-search": {
      "command": "python3",
      "args": ["-m", "gemini_search_mcp"],
      "env": { "GEMINI_API_KEY": "..." }
    }
  },
  "permissions": {
    "allow": ["mcp__gemini-search__*"]
  }
}
```
