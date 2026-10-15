# Plan: Install REA and connect it to Claude Code

## Context
User asked to run `npx rea-agents@latest setup`, show the plan for approval first, then verify.
REA ("Reverse Engineer Anything") is a local MCP server + CLI that gives coding agents
reverse-engineering tools. Nothing has been installed yet; research so far was read-only.

## Vetting result (supermatt:dependency-vetting)
Trust was traced from upstream to the install command, not from the registry back.
- Upstream README (github.com/morluto/rea) publishes `npx rea-agents setup` / `npx rea-agents@latest setup` and names the npm package `rea-agents`. Matches the user's command. [Certain]
- npm `rea-agents@6.5.0` (latest, modified 2026-10-14): repository `github.com/morluto/rea`, MIT, bins `rea` / `rea-agents` (match docs). [Certain]
- Published by GitHub Actions via npm trusted publisher (OIDC) with SLSA provenance attestation. [Certain]
- No `preinstall` / `install` / `postinstall` scripts. [Certain]
- Not yet run: `npm audit signatures` (needs an installed tree). Will run it after install as a check.

Residual risk: setup writes agent config that lets every Claude Code session launch REA and
call its tools, including runtime capture that executes target software with user permissions. [Likely, per third-party write-up + README warning]

## Environment facts
- Node v22.22.0, npm 10.9.4: satisfies REA's floor (22.x >= 22.19). [Certain]
- Working tree clean on branch `ccr-68ca4f45-mflczu`. Repo `.mcp.json` exists and will NOT be touched (user-scope setup does not write it). [Certain]
- `~/.claude.json` exists (56 KB); `~/.claude/settings.json` does not. [Certain]
- This is an ephemeral cloud container: the registration lands in `/root/.claude.json` and will not carry over to the user's laptop or later sessions. [Certain]

## Steps (after approval)
1. **Dry run, no changes:** `npx -y rea-agents@latest setup --client claude_code --dry-run`
   Expect status `planned`, exit 0. Show the exact planned file changes to the user.
2. **Apply, Claude Code only, user scope:** `npx -y rea-agents@latest setup --client claude_code --yes --json`
   - Writes `mcpServers.rea` into `~/.claude.json`
   - Installs bundled skill into `~/.claude/skills/reverse-engineer-anything`
   - Backs up first (`.rea.backup`, next to the edited file)
   - Not using `--all-detected`: avoids touching Codex/Cursor/Gemini configs.
   - Hopper install is optional and approval-gated; decline it (not needed for JS/Electron/web analysis).
3. **Verify:**
   - `npx -y rea-agents@latest doctor --client claude_code --skill --json` -> expect `healthy: true`, skill state installed, `installed_version` 6.5.0
   - `claude mcp list` -> `rea` listed (if `claude` CLI is available here)
   - `npm audit signatures` against the npx-cached package if feasible
   - `git status` in the repo -> still clean
4. **Report** results honestly. Note the current session needs a restart to expose REA's tools, so tool-level verification (calling an REA MCP tool) is not possible in this session. [Certain, per docs]

## Rollback
`claude mcp remove --scope user rea`, remove `~/.claude/skills/reverse-engineer-anything`,
or `rea uninstall` (add `--purge-data` to also drop `~/.rea`). Restore from `.rea.backup` if needed.

## Not doing
- No project-scope (`.mcp.json`) install; docs advise against it and it would dirty this repo.
- No commits or PR; nothing in the repo changes.
