# sentinel-skills

Claude Code skills for [Sentinel](https://github.com/buildsentinelhq/sentinel) — AI agent security on Solana.

## Skills

| Skill | Description |
|-------|-------------|
| `sentinel` | Runtime security skill — teaches your AI agent to scan inputs and simulate transactions before executing them |
| `sentinel-dev` | Developer integration guide — SDK API reference, configuration patterns, custom rule authoring |

## Install

### Claude Code Plugin (recommended)

```bash
/plugin marketplace add buildsentinelhq/sentinel-skills
/plugin install sentinel-skills
```

### npx skills

Uses [`npx skills`](https://github.com/vercel-labs/skills) — the open agent skills tool. No extra packages to install.

```bash
npx skills add buildsentinelhq/sentinel-skills
```

The interactive prompt will ask which skills you want and whether to install globally (all projects) or locally (this project only).

**Install a specific skill:**

```bash
npx skills add buildsentinelhq/sentinel-skills/skills/sentinel
npx skills add buildsentinelhq/sentinel-skills/skills/sentinel-dev
```

**Install locations:**

| Choice | Path | When to use |
|--------|------|-------------|
| Global | `~/.claude/plugins/` | Available in every Claude Code project |
| Local | `./.claude/plugins/` | This project only — commit to share with your team |

## What the skills do

### `sentinel`

Once installed, Claude Code uses this skill whenever it's working with an AI agent that touches Solana. It enforces a two-step security gate on every action:

1. `sentinel scan` on the user's instruction — blocks prompt injection, drain attempts, jailbreaks
2. `sentinel simulate` on the transaction — blocks policy violations before signing

Requires the `sentinel` CLI from [`sentinel`](https://github.com/buildsentinelhq/sentinel).

### `sentinel-dev`

A reference guide skill for developers integrating the SDK. Claude loads it when you ask about `@sentinelhq/core` — covers the full API, configuration patterns (rules-only, LLM+rules, production-strict), custom YAML rule authoring, error codes, and logging.

## Contributing

Skills live in `skills/<name>/SKILL.md`. To add a new skill:

1. Create `skills/your-skill/SKILL.md` with `name` and `description` in the frontmatter
2. Add any supporting files (YAML configs, templates, references) under `skills/your-skill/`
3. Register the path in `.claude-plugin/marketplace.json`
4. Open a PR

## License

MIT
