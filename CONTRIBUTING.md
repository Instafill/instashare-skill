# Contributing

The canonical skill source lives in [`skills/instashare/SKILL.md`](./skills/instashare/SKILL.md). The repo root also acts as a Claude Code plugin marketplace via [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json).

## Local development

1. Edit `skills/instashare/SKILL.md` (and any future supporting files in the same folder).
2. Test the skill in Claude Code by copying it into `~/.claude/skills/instashare/` or installing via `/plugin marketplace add /absolute/path/to/this/repo`.
3. Try the trigger phrases from the README in a real session and confirm the URL it returns works in a browser.

## Submitting to anthropics/skills

We also mirror this skill into the [anthropics/skills](https://github.com/anthropics/skills) repository so it shows up there. The directory layout under `skills/instashare/` already matches what that repo expects (`SKILL.md` + `LICENSE.txt`), so submitting an update means:

1. Fork `anthropics/skills`.
2. Copy `skills/instashare/` from this repo into `skills/instashare/` in the fork.
3. Add an entry to `.claude-plugin/marketplace.json` in that repo (under the `example-skills` or a new plugin).
4. Open a PR linking back to this repo as the canonical source.

## License

By contributing you agree your changes are licensed under Apache-2.0.
