# Claude Skills for Vercel

A collection of skills for [Claude.ai](https://claude.ai) and [Claude Code](https://claude.com/claude-code) for working with Vercel deployments.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### vercel-deploy

Deploy applications and websites to Vercel instantly. No authentication required.

**Use when:**
- "Deploy my app"
- "Deploy this to production"
- "Push this live"
- "Deploy and give me the link"

**Features:**
- No authentication required - works instantly
- Auto-detects 40+ frameworks from `package.json`
- Returns preview URL (live site) and claim URL (transfer ownership)
- Handles static HTML projects automatically
- Excludes `node_modules` and `.git` from uploads

**How it works:**
1. Packages your project into a tarball
2. Detects framework (Next.js, Vite, Astro, etc.)
3. Uploads to deployment service
4. Returns preview URL and claim URL

**Output:**
```
✓ Deployment successful!

Preview URL: https://skill-deploy-abc123.vercel.app
Claim URL:   https://vercel.com/claim-deployment?code=...
```

## Installation

### Claude Code

Use the `/install-skill` slash command to install directly from GitHub:

```
/install-skill https://github.com/vercel-labs/claude-skills/tree/main/skills/<skill-name>
```

For example, to install `vercel-deploy`:

```
/install-skill https://github.com/vercel-labs/claude-skills/tree/main/skills/vercel-deploy
```

Add `--personal` to install to `~/.claude/skills/` (available across all projects) or `--project` for `.claude/skills/` (project-specific, default).

See the [Claude Code Skills docs](https://code.claude.com/docs/en/skills) for more details.

### claude.ai

Add the skill to your project knowledge or paste the contents of `SKILL.md` into your conversation.

No CLI installation or authentication needed.

### Codex

Follow the [Codex skills guide](https://developers.openai.com/codex/skills/) and place the skill under `$CODEX_HOME/skills`:

```bash
# from the repo root
# defaults to ~/.codex if CODEX_HOME is unset
export CODEX_HOME="${CODEX_HOME:-$HOME/.codex}"
mkdir -p "$CODEX_HOME/skills"
cp -r skills/vercel-deploy "$CODEX_HOME/skills/"
```

Codex will auto-discover `SKILL.md` files in that directory on the next start.

### OpenCode

Copy the desired skill folder to your skills directory:

```bash
cp -r skills/vercel-deploy ~/.claude/skills/
```

OpenCode discovers skills from `~/.claude/skills/<name>/SKILL.md` automatically. See [OpenCode Skills docs](https://opencode.ai/docs/skills/) for more details.

## Usage

Skills are automatically available once installed. Simply ask Claude to deploy:

```
Deploy my app
```

```
Deploy this and give me the link
```

Claude will package your project, deploy it, and return both URLs.

## Skill Structure

Each skill contains:
- `SKILL.md` - Instructions for Claude
- `scripts/` - Helper scripts for automation

## Troubleshooting

### Network egress error on claude.ai

If deployment fails due to network restrictions, you need to allow Vercel domains:

1. Go to [claude.ai/admin-settings/capabilities](https://claude.ai/admin-settings/capabilities)
2. Add `*.vercel.com` to the allowed domains

## License

MIT
