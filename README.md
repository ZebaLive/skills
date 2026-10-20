# skills

ZebaLive's skills for Claude Code and Codex. Each plugin holds one skill, so you can install any skill on its own.

| Plugin | What it does |
| --- | --- |
| [`review-prs`](plugins/review-prs/skills/review-prs/SKILL.md) | Reviews one or more GitHub PRs and writes one validated review file, sorted by severity, in depthfirst style. Re-reviews only what changed after fixes. |

## Install in Claude Code

```bash
claude plugin marketplace add ZebaLive/skills
claude plugin install review-prs@zeba-skills
```

Inside a session, `/plugin marketplace add ZebaLive/skills` and `/plugin install review-prs@zeba-skills` do the same.

## Install in Codex

```bash
codex plugin marketplace add ZebaLive/skills
codex plugin add review-prs@zeba-skills
```

## Update

```bash
claude plugin marketplace update zeba-skills
codex plugin marketplace upgrade
```

## Add a skill

1. Create `plugins/<name>/skills/<name>/SKILL.md`.
2. Copy `plugins/review-prs/.claude-plugin/plugin.json` to `plugins/<name>/.claude-plugin/plugin.json`. Change the name and description.
3. Add an entry for the plugin to `.claude-plugin/marketplace.json`.
4. Run `claude plugin validate .` and fix every error.

Codex reads the same `.claude-plugin/` files, so one layout serves both tools.

## Layout

```
.claude-plugin/marketplace.json        marketplace "zeba-skills", one entry per plugin
plugins/<name>/.claude-plugin/plugin.json
plugins/<name>/skills/<name>/SKILL.md  the skill
```
