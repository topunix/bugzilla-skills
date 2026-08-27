# bugzilla-skills

Claude Skills for working on Bugzilla and the Bugzilla 6 (Harmony) codebase.

Each skill is a folder containing a `SKILL.md` file plus reference material.
Claude loads a skill when the work matches its description, so conventions
that normally live in a maintainer's head are applied consistently across
triage, patch review, and migration work.

Targets Bugzilla 5.0 and later, including the Harmony branch.

## Skills

| Skill | Use for |
| ----- | ------- |
| `bugzilla-triage` | Dependency tree analysis, duplicate detection, status and resolution semantics |
| `perl-patch-review` | Bugzilla Perl conventions, `Bugzilla::Object` patterns, validators, taint handling |
| `schema-migration-review` | `Bugzilla/Install/DB.pm` patterns, idempotency, cross database portability |
| `template-toolkit-review` | Template conventions, filter discipline, hooks, cache invalidation |

## Installation

Copy the skill folders you want into `.claude/skills/` in your Bugzilla
checkout, or into `~/.claude/skills/` to make them available in every project.

## Sourcing policy

Every convention asserted in these skills is drawn from public sources: the
Bugzilla documentation at bugzilla.readthedocs.io, the public source tree at
github.com/bugzilla/harmony, and public project guidelines. Each skill cites
the source for the rules it states. Nothing here is derived from private or
client specific material, and unverifiable conventions are omitted rather
than guessed.

## Status

Early. Skills are added as they are validated against the public codebase.

## License

See LICENSE.

## Not affiliated

This is an independent project and is not affiliated with or endorsed by the
Bugzilla project or Mozilla.# bugzilla-skills