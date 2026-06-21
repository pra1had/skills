# Skills

Generic, reusable Claude Code skills, kept in git and symlinked into each
vault/project at `.claude/skills/<name>`. **No personal data lives here** —
per-vault config and data stay in the vault and travel via that tool's own sync.

## Skills

- `tasks/` — day-to-day task management for an Obsidian vault. Design reference:
  `tasks/SPEC.md`; runnable skill: `tasks/SKILL.md`.

## New-machine recovery

The skill logic and the vault recover independently.

1. **Skill logic:** `git clone <this-repo-url> ~/work/Skills`
2. **Symlink (machine-local, not synced):** from the vault root, re-create the
   link Claude Code reads:
   ```bash
   ln -s ../../../Skills/tasks <vault>/.claude/skills/tasks
   ```
3. **Vault data:** the vault (including `diary/` notes, `diary/_tasks-config.md`,
   and `diary/_template.md`) returns via Obsidian Sync — nothing to do here.
