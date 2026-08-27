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
| [`bugzilla-triage`](skills/bugzilla-triage/SKILL.md) | Dependency tree analysis, duplicate detection, status and resolution semantics |
| [`perl-patch-review`](skills/perl-patch-review/SKILL.md) | Bugzilla Perl conventions, `Bugzilla::Object` patterns, validators, taint handling |
| [`schema-migration-review`](skills/schema-migration-review/SKILL.md) | `Bugzilla/Install/DB.pm` patterns, idempotency, cross database portability |
| [`template-toolkit-review`](skills/template-toolkit-review/SKILL.md) | Template conventions, filter discipline, hooks, cache invalidation |

### Trigger conditions

Claude selects a skill by matching the task at hand against its
`description` frontmatter. These are the situations each skill is written
to trigger on:

- **`bugzilla-triage`** — reading or building a bug's Depends On/Blocks
  tree, deciding whether a new report is a duplicate of an existing bug,
  judging whether a bug is a "blocker" versus routine severity, or picking
  a legal status/resolution transition (e.g. is `IN_PROGRESS → VERIFIED`
  even legal). Not for writing code — this is a data-model/workflow skill.

- **`perl-patch-review`** — reviewing or writing a Perl diff against
  `Bugzilla/*.pm`, a `*.cgi` script, or a `Bugzilla::Object` subclass.
  Triggers on questions like "does this follow Bugzilla's style", "will
  this pass `perl -c`/the test suite", or "is this validator/taint pattern
  correct".

- **`schema-migration-review`** — reviewing or writing a schema change:
  anything touching `Bugzilla/Install/DB.pm`, `Bugzilla/DB/Schema.pm`, or
  calls to `bz_add_column`/`bz_column_info`/`bz_alter_column`/
  `bz_add_index`. Triggers on "will this migration run safely twice" and
  "is this portable across MySQL/PostgreSQL/SQLite".

- **`template-toolkit-review`** — reviewing or writing a `.tmpl` file.
  Triggers on "does this template escape user data correctly" (`FILTER
  html` discipline), hook-point (`Hook.process`) correctness, and "why
  isn't my template change showing up" (compiled template cache).

## Installation

Copy the skill folders you want into `.claude/skills/` in your Bugzilla
checkout, or into `~/.claude/skills/` to make them available in every
project:

```sh
cp -r skills/bugzilla-triage skills/perl-patch-review \
      skills/schema-migration-review skills/template-toolkit-review \
      ~/.claude/skills/
```

Or copy the whole set:

```sh
cp -r skills/* ~/.claude/skills/
```

Each skill also works standalone — copy just the one folder you need if you
don't want the full set installed.

## Sourcing policy

Every convention asserted in these skills is drawn from public sources: the
Bugzilla documentation at bugzilla.readthedocs.io, the public source tree at
github.com/bugzilla/harmony, and public project guidelines
(bugzilla.org, github.com/bugzilla/bugzilla docs). Each skill's `SKILL.md`
cites its sources inline, and each skill's `reference/` folder has the full
quotes and links behind those citations. Nothing here is derived from
private or client specific material, and unverifiable conventions are
omitted rather than guessed. Where Harmony's actual code has diverged from
older/general Bugzilla folklore (e.g. tabs vs. spaces), the skill follows
what the current public source tree actually enforces, not the folklore.

## Status

Early. Skills are added as they are validated against the public codebase.

## License

See LICENSE.

## Not affiliated

This is an independent project and is not affiliated with or endorsed by the
Bugzilla project or Mozilla.
