# ultrapack

Custom Claude Code skill pack built by merging [obra/superpowers](https://github.com/obra/superpowers) and [Anthropic feature-dev](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev).

## Purpose

Own our workflow skills instead of depending on upstream packs that update on someone else's schedule. Take the parts of superpowers and feature-dev that actually help, drop the rest, freeze the behavior.

## Source material (read-only references)

- `superpowers/` — full clone of obra/superpowers. Skills in `superpowers/skills/`
- `claude-code-plugins/plugins/feature-dev/` — sparse checkout of Anthropic's feature-dev plugin (agents + commands)

Do not modify these directories. They are upstream snapshots for reference only.

## Design principles

- Fork-and-own: our skills live in this repo, not as a dependency on upstream
- Minimal: include only skills we actually use
