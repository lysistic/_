<!-- ARIS:BEGIN -->
## ARIS Skill Scope
ARIS skills installed in this project: 83 skills + `shared-references`.
Source: https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep (commit `2132036`), installed as plain copies (not symlinks) so they survive ephemeral cloud sessions.
Skills live under `.claude/skills/`; helper scripts live under `.aris/tools/` (skills resolve helpers there first).
For ARIS workflows, prefer the project-local skills under `.claude/skills/` over global skills.
Update by re-copying `skills/*` (except `skills-codex*`) and `tools/` from a fresh clone of the upstream repo.
<!-- ARIS:END -->
