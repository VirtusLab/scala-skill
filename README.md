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
