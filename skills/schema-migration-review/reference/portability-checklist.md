# Cross-database portability — sources

Source: [`t/013db_portability.t`](https://github.com/bugzilla/harmony/blob/main/t/013db_portability.t)
in [`bugzilla/harmony`](https://github.com/bugzilla/harmony) (github.com),
current `main` branch. This is an actual gating test — it scans Perl
source and script files (excluding the `Bugzilla/DB/` driver modules
themselves, which are allowed to speak vendor SQL) and fails on use of
these functions:

| Forbidden call | Required `Bugzilla::DB` method |
| --- | --- |
| `UNIX_TIMESTAMP()` | `$dbh->sql_date_to_epoch(...)` |
| `DATE_FORMAT()` | `$dbh->sql_date_format(...)` |
| `CONCAT()` | `$dbh->sql_string_concat(...)` |
| `POSITION()` / `INSTR()` / `LOCATE()` | `$dbh->sql_position(...)` or `$dbh->sql_iposition(...)` |
| `GROUP_CONCAT()` | `$dbh->sql_group_concat(...)` |

The test reports the offending filename and line number on failure. The
rationale, per how the check is structured: centralizing database
compatibility logic into `Bugzilla::DB` helper methods lets the same
migration/query code run unmodified against MySQL, PostgreSQL, and SQLite
— each `sql_*` method is implemented per-database in the corresponding
`Bugzilla::DB::*` subclass, so application code never branches on which
database it's running against.

## Review use

When reviewing a schema or query change:

1. Grep the diff for the forbidden function names above (case-insensitive,
   they can appear inside a quoted SQL string).
2. Any hit outside `Bugzilla/DB/*.pm` is a portability regression — replace
   it with the corresponding `sql_*` call on `$dbh`, sourced from the
   installation's actual database handle rather than assumed.
3. If a needed operation has no existing `sql_*` wrapper, that's a signal
   to add one to `Bugzilla::DB` (and its per-database subclasses) rather
   than inlining vendor SQL at the call site — matches the project's own
   stated intent that `Bugzilla::DB` is where vendor differences live.
