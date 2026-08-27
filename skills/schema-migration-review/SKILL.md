---
name: schema-migration-review
description: Use when reviewing or writing a database schema change for Bugzilla 5.0+/Harmony - anything touching Bugzilla/Install/DB.pm, Bugzilla/DB/Schema.pm, or code calling bz_add_column/bz_column_info/bz_alter_column/bz_add_index. Covers idempotent upgrade-script patterns, the abstract-schema type system, and the specific DB-specific SQL functions Harmony's own test suite forbids for MySQL/PostgreSQL/SQLite portability. Trigger for "will this migration run safely twice", "is this portable across databases", or schema/DDL patch review.
---

# Bugzilla Schema Migration Review

Bugzilla upgrade code runs unconditionally on every `checksetup.pl`/install
invocation, against installations at every prior schema version — so every
migration in `Bugzilla::Install::DB` must be safe to run against a database
that already has the change applied. Portability across MySQL, PostgreSQL,
and SQLite is enforced by an actual test (`t/013db_portability.t`), not just
convention. Sources are in `reference/`; nothing here is asserted without a
citation to the public `bugzilla/harmony` source tree.

## Idempotency pattern

The standard pattern in `Bugzilla::Install::DB` is: check whether the
change already exists, and only apply it if not.

```perl
# Column
if (!$dbh->bz_column_info('fielddefs', 'obsolete')) {
  $dbh->bz_add_column('fielddefs', 'obsolete',
    {TYPE => 'BOOLEAN', NOTNULL => 1, DEFAULT => 'FALSE'});
}

# Index
if (!$dbh->bz_index_info('email_rates', 'email_rates_message_ts_idx')) {
  $dbh->bz_add_index('email_rates', 'email_rates_message_ts_idx',
    ['message_ts']);
}
```

When reviewing a new migration function:
- A bare `$dbh->bz_add_column(...)` with no preceding `bz_column_info`
  check is a review finding — it will error (or silently duplicate,
  depending on driver) on a second run.
- `bz_alter_column` is self-guarding: per its POD, "If the new type is the
  same as the old type, the function returns without changing anything," so
  it doesn't need a manual existence check the way `bz_add_column` does.
- `bz_drop_column`, per its POD, "returns without doing anything" if the
  column doesn't exist — also safe to call unconditionally.
- `bz_add_column` **cannot** add a `NOT NULL` column with no `DEFAULT`,
  because "the database won't know what to set all the NULL values to" on
  existing rows — a migration adding a `NOTNULL => 1` column must also
  supply `DEFAULT`.

## Abstract schema over DB-specific SQL

Column types are declared via `Bugzilla::DB::Schema`'s `ABSTRACT_SCHEMA`
vocabulary (`INT1`, `INT2`, `INT3`, `INT4`, `MEDIUMSERIAL`, `BIGSERIAL`,
`MEDIUMTEXT`, `LONGBLOB`, `DATETIME`, `BOOLEAN`, etc.), never a raw
MySQL/Postgres/SQLite column type string. Per its POD, this module "should
be considered package-private to the `Bugzilla::DB` module" — a migration
that constructs SQL type strings directly instead of going through
`ABSTRACT_SCHEMA`/`bz_add_column` is bypassing the one place that knows how
to translate a type per-database.

The same discipline applies to functions, not just types.
`t/013db_portability.t` scans Perl source and **fails the build** on any of
these raw SQL function calls outside `Bugzilla/DB/`:

| Forbidden (DB-specific) | Required (`Bugzilla::DB` method) |
| --- | --- |
| `UNIX_TIMESTAMP()` | `$dbh->sql_date_to_epoch(...)` |
| `DATE_FORMAT()` | `$dbh->sql_date_format(...)` |
| `CONCAT()` | `$dbh->sql_string_concat(...)` |
| `POSITION()` / `INSTR()` / `LOCATE()` | `$dbh->sql_position(...)` / `sql_iposition(...)` |
| `GROUP_CONCAT()` | `$dbh->sql_group_concat(...)` |

Any of these appearing literally in a patch's SQL (outside the DB driver
modules themselves) is a portability bug, not a style nit — it will pass on
whichever database the author tested against and break on the others.

## Review checklist

1. Does every `bz_add_column`/`bz_add_index` call have a guarding
   `bz_column_info`/`bz_index_info` check, or is it one of the
   self-guarding calls (`bz_alter_column`, `bz_drop_column`)?
2. Does every new `NOTNULL` column also specify `DEFAULT`?
3. Are all types expressed in `ABSTRACT_SCHEMA` vocabulary, not a raw
   database-specific type string?
4. Does the SQL avoid the forbidden vendor-specific functions listed above,
   using the `Bugzilla::DB` portability methods instead?
5. Is the new migration function actually invoked from the upgrade driver
   (added to the sequence `Bugzilla::Install::DB` runs), not just defined?

See `reference/db-abstraction.md` and `reference/portability-checklist.md`
for full source quotes.
