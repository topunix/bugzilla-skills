# Bugzilla::Object and taint idioms — sources

Source: [`Bugzilla/Object.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Object.pm)
and [`Bugzilla/Util.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Util.pm)
in [`bugzilla/harmony`](https://github.com/bugzilla/harmony) (github.com).

## Validators

```perl
use constant VALIDATORS => {};
use constant UPDATE_VALIDATORS => {};
use constant VALIDATOR_DEPENDENCIES => {};
```

POD from `Bugzilla/Object.pm`: "Validators are called both by `create()`
and `set()`. When they are called by `create()`, the first argument will be
the name of the class ... When they are called by `set()`, the first
argument will be a reference to the current object."

On `VALIDATOR_DEPENDENCIES`: "During `create()` and `set_all()`, validators
are normally called in a somewhat-random order. If you need one field to be
validated and set before another field, this constant is how you do it, by
saying that one field 'depends' on the value of other fields."

Example validator (illustrative, from the same file):

```perl
sub check_boolean { return $_[1] ? 1 : 0 }

sub check_time {
  my ($invocant, $value, $field, $params, $allow_negative) = @_;
  my $current = blessed($invocant) ? $invocant->{$field} : 0;
  $current ||= 0;
  $value = trim($value) || 0;
  _validate_time($value, $field, $allow_negative);
  return $value;
}
```

Dependency-ordered validation is implemented with `_sort_by_dep`, which
partitions fields into those with declared dependencies and those without,
and inserts dependent fields after their dependencies:

```perl
sub _sort_by_dep {
  my ($invocant, @fields) = @_;
  my $dependencies = $invocant->VALIDATOR_DEPENDENCIES;
  my ($has_deps, $no_deps) = part { $dependencies->{$_} ? 0 : 1 } @fields;
  my @result = sort @{$no_deps || []};
  foreach my $field (sort @{$has_deps || []}) {
    if (!grep { $_ eq $field } @result) {
      _insert_dep_field($field, $has_deps, $dependencies, \@result);
    }
  }
  return @result;
}
```

## Column declarations

```perl
use constant DB_COLUMNS => [];
use constant UPDATE_COLUMNS => [];
```

`_get_db_columns()` resolves the actual column set, with caching and hook
processing, consulting `DYNAMIC_COLUMNS` to decide whether to query the
information schema or use the predefined list.

## Taint handling inside Bugzilla::Object

`_load_from_db` and `match()` both detaint numeric inputs before using them
in SQL:

```perl
detaint_natural($id) || ThrowCodeError('param_must_be_numeric',
  {function => $class . '::_load_from_db'});

detaint_natural($value) or ThrowCodeError('param_must_be_numeric',
  {param => 'LIMIT', function => "${class}::match"});

detaint_signed($offset) or ThrowCodeError('param_must_be_numeric',
  {param => 'OFFSET', function => "${class}::match"});
```

## `detaint_natural` / `detaint_signed`

From `Bugzilla/Util.pm`:

```perl
sub detaint_natural {
  my $match = $_[0] =~ /^(\d+)$/a;
  $_[0] = $match ? int($1) : undef;
  return (defined($_[0]));
}

sub detaint_signed {
  my $match = $_[0] =~ /^([-+]?\d+)$/a;
  $_[0] = $match ? int($1) : undef;
  return (defined($_[0]));
}
```

POD for both, verbatim: "This routine detaints a [natural number / signed
integer]. It returns a true value if the value passed in was a valid
[natural number / signed integer], else it returns false. You **MUST**
check the result of this routine to avoid security holes."

## "Trick taint" idiom

From [`importxml.pl`](https://github.com/bugzilla/harmony/blob/main/importxml.pl):

```perl
BEGIN {
  require File::Basename;
  my $dir = $0;
  $dir =~ /(.*)/;
  $dir = $1;    # trick taint
  chdir(File::Basename::dirname($dir));
}
```

A regex capture into `$1` is used to launder `$0` before it's used in a
`chdir`, since a script may be invoked from an arbitrary/untrusted working
directory (e.g. by a mail transfer agent). This idiom untaints without
validating the *content* — prefer `detaint_natural`/`detaint_signed` for
numeric data, which validate the format as a condition of untainting.

## Shebang/pragma evidence (no blanket `-T`)

Verified directly against `main`:

- [`enter_bug.cgi`](https://github.com/bugzilla/harmony/blob/main/enter_bug.cgi):
  shebang `#!/usr/bin/env perl`, followed by `use strict; use warnings;` —
  no `-T`.
- [`Bugzilla.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla.pm):
  `use 5.14.0; use strict; use warnings;` — no `taint`/`-T`/`PERL_TAINT`
  reference anywhere in the file.

This is stated narrowly: it confirms these two files don't enable taint
mode via shebang. It is not a claim that no code path in Harmony runs under
`-T` (e.g. via a wrapper), only that the primary CGI entry point inspected
does not request it directly.
