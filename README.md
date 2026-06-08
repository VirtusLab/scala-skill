# Scala Skills

A collection of [Scala](https://www.scala-lang.org) skills for [Claude
Code](https://claude.ai/claude-code) and [Codex](https://openai.com/codex/).
Written by AI, for AIs and humans alike.

## Available skills

- **[direct-style-scala](direct-style-scala/skills/direct-style-scala/)** —
  Scala coding style, tooling, and functional programming guidance, with
  dedicated sections on direct-style Scala, [Ox](https://ox.softwaremill.com)
  structured concurrency, and synchronous
  [Tapir](https://tapir.softwaremill.com).

## Installation

### Claude Code

#### Add the marketplace

```
/plugin marketplace add virtuslab/scala-skill
```

#### Install the plugin-provided skill

```
/plugin install direct-style-scala@virtuslab-scala-skill
```

#### Manual installation

```bash
git clone https://github.com/virtuslab/scala-skill.git /tmp/scala-skill
mkdir -p ~/.claude/skills
cp -r /tmp/scala-skill/direct-style-scala/skills/direct-style-scala ~/.claude/skills/
```

### Codex

Install the Codex plugin marketplace:

```bash
codex plugin marketplace add virtuslab/scala-skill
```

Then open Codex and install **direct-style-scala** from `/plugins`.

If you added the marketplace before, refresh it first:

```bash
codex plugin marketplace upgrade virtuslab-scala-skill
```

For local development, add a checkout as a marketplace instead:

```bash
git clone https://github.com/virtuslab/scala-skill.git /tmp/scala-skill
codex plugin marketplace add /tmp/scala-skill
```

### Cursor, OpenCode, and Pi

These editors load [Agent Skills](https://agentskills.io) (the `SKILL.md`
format) directly. All three discover skills from the vendor-neutral
`~/.agents/skills/` directory, so a single copy makes the skill available in
each of them:

```bash
git clone https://github.com/virtuslab/scala-skill.git /tmp/scala-skill
mkdir -p ~/.agents/skills
cp -r /tmp/scala-skill/direct-style-scala/skills/direct-style-scala ~/.agents/skills/
```

Restart the editor; the agent loads the skill automatically when a task looks
relevant. You can also trigger it explicitly: `/direct-style-scala` in Cursor's
Agent chat, or `/skill:direct-style-scala` in Pi.

Each tool also has its own native skills directory if you prefer to scope the
install per tool — use it instead of `~/.agents/skills/` above:

- **[Cursor](https://cursor.com/docs/skills)**: `~/.cursor/skills/` (global) or
  `.cursor/skills/` (per project)
- **[OpenCode](https://opencode.ai/docs/skills/)**:
  `~/.config/opencode/skills/` (global) or `.opencode/skills/` (per project)
- **[Pi](https://pi.dev/docs/latest/skills)**: `~/.pi/agent/skills/` (global) or
  `.pi/skills/` (per project)

For per-project use, copy the skill into the project-level `.agents/skills/`
directory instead, and commit it alongside your code.

#### Staying up to date

The `cp` above installs a snapshot — it does not auto-update. None of these
editors poll a remote for skill updates. To stay current across all three at
once, clone the repository once and symlink the skill into the skills
directory, then `git pull` to refresh:

```bash
git clone https://github.com/virtuslab/scala-skill.git ~/scala-skill
ln -s ~/scala-skill/direct-style-scala/skills/direct-style-scala ~/.agents/skills/direct-style-scala
# update later (refreshes Cursor, OpenCode, and Pi together):
git -C ~/scala-skill pull
```

Wrap the `git pull` in a cron job or shell-login hook if you want it to run
automatically.
