# DB abstraction and idempotency — sources

All from the public [`bugzilla/harmony`](https://github.com/bugzilla/harmony)
repository (github.com), current `main` branch.

## `Bugzilla::Install::DB` idempotent patterns

Source: [`Bugzilla/Install/DB.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Install/DB.pm).
The module comment notes: "NOTE: This package may 'use' any modules that it
likes, localconfig is available, and params are up to date" — i.e. by the
time these functions run, the install is otherwise fully bootstrapped, so a
migration function can assume normal Bugzilla API availability.

Representative idempotent patterns actually in the file:

```perl
if (!$dbh->bz_column_info('fielddefs', 'obsolete')) {
    $dbh->bz_add_column('fielddefs', 'obsolete',
      {TYPE => 'BOOLEAN', NOTNULL => 1, DEFAULT => 'FALSE'});
}

if (!$dbh->bz_index_info('email_rates', 'email_rates_message_ts_idx')) {
    $dbh->bz_add_index('email_rates', 'email_rates_message_ts_idx',
      ['message_ts']);
}

$dbh->bz_alter_column('bugs', 'bug_status',
  {TYPE => 'varchar(64)', NOTNULL => 1});
```

The module is organized as a chronological sequence of upgrade functions,
each responsible for bringing the schema from one historical state forward
— every function in that sequence runs on every `checksetup.pl` execution,
including against a database that's already current, which is why the
guard-before-add pattern is load-bearing rather than defensive style.

## `Bugzilla::DB` POD for the schema-modification methods

Source: [`Bugzilla/DB.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/DB.pm)
POD, quoted directly:

- **`bz_add_column`**: "Adds a new column to a table in the database.
  Prints out a brief statement that it did so, to stdout. Note that you
  cannot add a NOT NULL column that has no default -- the database won't
  know what to set all the NULL values to."
- **`bz_column_info`**: "Get abstract column definition." Returns "An
  abstract column definition for that column. If the table or column does
  not exist, we return `undef`."
- **`bz_alter_column`**: "Changes the data type of a column in a table.
  Prints out the changes being made to stdout. If the new type is the same
  as the old type, the function returns without changing anything."
- **`bz_drop_column`**: "Removes a column from a database table. If the
  column doesn't exist, we return without doing anything. If we do
  anything, we print a short message to `stdout` about the change."

The module's overall POD purpose statement: functions in this module "allow
creation of a database handle to connect to the Bugzilla database" and the
module "contains methods extending the returned handle with functionality
which is different between databases allowing for easy customization for
particular database via inheritance" — i.e. `Bugzilla::DB` is explicitly the
one place vendor differences are meant to live, via per-database
subclasses, not scattered inline SQL.

## `Bugzilla::DB::Schema` abstract types

Source: [`Bugzilla/DB/Schema.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/DB/Schema.pm)
POD, quoted directly:

> "The abstract database schema structure consists of a hash reference in
> which each key is the name of a table in the Bugzilla database. The value
> for each key is a hash reference containing the keys `FIELDS` and
> `INDEXES` which in turn point to array references containing information
> on the table's fields and indexes."

> "A field hash reference must contain the key `TYPE`. Optional field keys
> include `PRIMARYKEY`, `NOTNULL`, and `DEFAULT`."

> "This module implements an object-oriented, abstract database schema. It
> should be considered package-private to the `Bugzilla::DB` module."

Abstract types observed in the schema include `INT1`, `INT2`, `INT3`,
`INT4`, `MEDIUMSERIAL`, `BIGSERIAL`, `LONGBLOB`, `MEDIUMTEXT`, `DATETIME`,
and `BOOLEAN`. A per-database `_adjust_schema()` implementation is
responsible for turning these into the vendor-specific SQL type at DDL
generation time — application/migration code should never need to know
what `BOOLEAN` compiles to on a given backend.
