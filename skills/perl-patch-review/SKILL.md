---
name: perl-patch-review
description: Use when reviewing or writing a Perl patch against Bugzilla 5.0+/Harmony - a change touching Bugzilla/*.pm modules, a *.cgi script, or Bugzilla::Object subclasses. Covers Harmony's actual coding-style enforcement (perltidy/perlcritic config, tabs-vs-spaces, shebang and pragma rules from the t/ test suite), Bugzilla::Object validator/dependency patterns, and taint-safety idioms (detaint_natural/detaint_signed). Trigger for patch review, "does this follow Bugzilla conventions", or "will this pass perl -c / the test suite" questions.
---

# Bugzilla Perl Patch Review

Reviews should check two independent things: does the patch pass Harmony's
own automated style/compile checks, and does it follow the
`Bugzilla::Object` conventions for data classes. Every rule here is sourced
directly from the `bugzilla/harmony` repo (config files and `t/*.t` test
sources) or the public Bugzilla Developer's Guide — see `reference/` for the
full quotes. Don't apply a rule from general Perl folklore (e.g. "Bugzilla
uses tabs") unless it's confirmed below; Harmony's actual enforced style has
diverged from some older Bugzilla lore.

## Style and gating checks

Harmony enforces style with a real test suite (`t/002goodperl.t`,
`t/005whitespace.t`, `critic.t`, `011pod.t`), not just convention. A patch
should satisfy all of:

- **No tabs.** `t/005whitespace.t` fails any file containing a tab
  character ("contains tabs --WARNING"). Bugzilla/Harmony code is
  **space-indented**, not tab-indented — this reverses the "Bugzilla uses
  tabs" folklore from much older releases.
- **2-space indent**, LF line endings, final newline — from `.editorconfig`,
  applied to `.pl`/`.pm`/`.cgi`/`.t` and to templates/CSS/JS/YAML.
- **perltidy** with `-pbp` (Perl Best Practices) as the base profile,
  `-i=2 -ci=2` (2-space indent/continuation), 80-column lines, high
  paren/brace/bracket tightness (`.perltidyrc`). Run perltidy with the
  repo's own `.perltidyrc` before proposing a diff, don't hand-format.
- **perlcritic** at `severity = 1` with the `freenode`/`core`/`certrec`/
  `certrule`/`performance`/`security` themes, plus repo-specific
  policy overrides in `.perlcriticrc` (e.g. trailing commas required,
  parens forbidden around builtins, no unbraced filehandles with `print`
  is *not* enforced, punctuation vars restricted to
  `$@ $! $/ $^O $^V`).
- **Shebang discipline** (`t/002goodperl.t`): an executable Perl file's
  shebang must be exactly `#!/usr/bin/env perl`; `.pm` files must have *no*
  shebang at all.
- **`use 5.14.0; use strict; use warnings;`** on every file (satisfied
  automatically if the file uses Moo/Role::Tiny/Mojo, which pull these in).
- **No inline error strings.** `t/002goodperl.t` flags any
  `Throw*Error(...)` call whose message isn't a localization tag — error
  text belongs in the template message catalog, not as a literal string in
  Perl.
- **No `=> $cgi->param(...)` hash-value pattern** — flagged directly by
  `t/002goodperl.t` as an insecure CGI-parameter idiom (multi-valued params
  silently becoming a list where a scalar was expected).

See `reference/style-and-tooling.md` for the exact config/test contents.

## `Bugzilla::Object` patterns

Data classes extend `Bugzilla::Object` and are expected to declare, as
class constants:

- `DB_COLUMNS` / `UPDATE_COLUMNS` — which columns exist and which are
  settable via `update()`.
- `VALIDATORS` / `UPDATE_VALIDATORS` — a `{ field => \&check_sub }` map;
  validators run during `create()` and `set()`. On `create()` the first
  argument is the class name; on `set()` it's the object reference — a
  validator that assumes one calling convention over the other is a bug.
- `VALIDATOR_DEPENDENCIES` — validators normally run in unspecified order;
  if field A's validator needs field B already validated, declare that
  dependency here rather than reading `$params->{B}` and hoping it ran
  first.

When reviewing a new field/validator, check that a validator either exists
for the field or that its absence is deliberate (e.g. a computed column).

## Taint-safety idioms

Harmony has largely moved primary CGI entry points away from `-T` in the
shebang (e.g. `enter_bug.cgi` and `Bugzilla.pm` both use
`#!/usr/bin/env perl` / no shebang plus `use strict; use warnings;`, with no
`-T`), but taint-safe idioms are still load-bearing in the codebase:

- `Bugzilla::Util::detaint_natural($value)` / `detaint_signed($value)`
  validate and untaint a number in place, returning true/false — POD is
  explicit: **"You MUST check the result of this routine to avoid security
  holes."** A call to `detaint_natural($id)` whose return value is discarded
  is a review finding, not a style nit.
- `Bugzilla::Object::_load_from_db` and `match()` call `detaint_natural`/
  `detaint_signed` on any numeric id/LIMIT/OFFSET before it reaches SQL —
  follow the same pattern for any new numeric input built into a query.
- The "trick taint" idiom (capture a value through a regex to launder it)
  still appears in standalone scripts, e.g. `importxml.pl`:
  ```perl
  my $dir = $0;
  $dir =~ /(.*)/;
  $dir = $1;    # trick taint
  ```
  Treat an unexplained bare regex-capture-into-self as this idiom, not dead
  code — but flag a *new* use of it in review; prefer `detaint_natural`/
  `detaint_signed` for numeric data, since they validate as well as
  untaint, where "trick taint" only launders.

See `reference/objects-validators-taint.md` for source snippets.
