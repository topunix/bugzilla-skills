# Status and resolution semantics — sources

## Default status workflow

Source: `Bugzilla::Install::STATUS_WORKFLOW` in
[`Bugzilla/Install.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Install.pm)
(github.com/bugzilla/harmony), which seeds the `status_workflow` table on
install and is enforced at runtime by `Bugzilla::Bug::_check_bug_status` in
[`Bugzilla/Bug.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Bug.pm):

```perl
use constant STATUS_WORKFLOW => (
  [undef,         'UNCONFIRMED'],
  [undef,         'CONFIRMED'],
  [undef,         'IN_PROGRESS'],
  ['UNCONFIRMED', 'CONFIRMED'],
  ['UNCONFIRMED', 'IN_PROGRESS'],
  ['UNCONFIRMED', 'RESOLVED'],
  ['CONFIRMED',   'IN_PROGRESS'],
  ['CONFIRMED',   'RESOLVED'],
  ['IN_PROGRESS', 'CONFIRMED'],
  ['IN_PROGRESS', 'RESOLVED'],
  ['RESOLVED',    'UNCONFIRMED'],
  ['RESOLVED',    'CONFIRMED'],
  ['RESOLVED',    'VERIFIED'],
  ['VERIFIED',    'UNCONFIRMED'],
  ['VERIFIED',    'CONFIRMED'],
);
```

Each pair is `[from_status_or_undef, to_status]`; `undef` means that status
is a legal status for a newly-filed bug. This is the *default* install
workflow — an individual installation's admin can add extra transitions
(e.g. NEW/ASSIGNED/REOPENED still exist as legacy statuses on some
installations), so treat this as the baseline, not a universal guarantee,
when working against a specific Bugzilla instance.

`Bugzilla::Bug::_check_bug_status` reads the transitions actually configured
for the instance (`$invocant->statuses_available`) and throws
`illegal_bug_status_transition` if the requested change isn't in that list.
The same method also:
- clears the resolution when moving to a non-closed status, and requires one
  when moving to a closed status,
- resets `everconfirmed` based on whether the new status is `UNCONFIRMED`,
- can require a target milestone on transition to `IN_PROGRESS` if the
  `musthavemilestoneonaccept` parameter is enabled.

## Resolution values

**The Bugzilla documentation does not define the resolution values.**
`docs/en/rst/using/understanding.rst` says of Status and Resolution: "The
different possible values for Status and Resolution on your installation
should be documented in the context-sensitive help for those items." So
there is no doc page to cite for a canonical list — it is installation-
specific by design.

What can be cited precisely is the set Bugzilla *ships* and seeds into a
fresh database. Source: `ENUM_DEFAULTS` in
[`Bugzilla/DB.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/DB.pm)
(github.com/bugzilla/harmony), which `bz_enum_initial_values` returns and
`bz_populate_enum_tables` writes into the enum tables at install time:

```perl
resolution => ["", "FIXED", "INVALID", "WONTFIX", "DUPLICATE", "WORKSFORME"],
```

That is the complete shipped default set: five named resolutions plus the
empty string (an open bug carries no resolution). For comparison, the same
constant seeds:

```perl
bug_status =>
  ["UNCONFIRMED", "CONFIRMED", "IN_PROGRESS", "RESOLVED", "VERIFIED"],
bug_severity =>
  ['blocker', 'critical', 'major', 'normal', 'minor', 'trivial', '--'],
```

Commonly understood meanings of the five, which are *not* quoted from a
Bugzilla doc page and are given here as description rather than citation:
FIXED (a fix has been checked in), INVALID (the problem described is not a
bug), WONTFIX (a real bug that will not be fixed), DUPLICATE (a duplicate of
an existing report; requires a target bug id — see
`reference/dependencies-and-duplicates.md`), WORKSFORME (could not be
reproduced with the information given).

Two cautions for triage:

- **`MOVED` is not a Harmony default.** It is absent from `ENUM_DEFAULTS`.
  Older Bugzilla releases and many long-lived installations carry it, along
  with values like `LATER` and `REMIND` (both visible in a legacy chart-data
  migration in `Bugzilla/Install/DB.pm`), and sites frequently add their own
  such as `NOTABUG`/`NOTOURBUG`/`INCOMPLETE`. None of these are guaranteed
  on a default 5.0+/Harmony install.
- Because resolutions are admin-configurable, confirm against the actual
  installation before asserting that a given resolution exists or what it
  means locally.

## Severity vs. dependency "Blocks"

Source: Bugzilla documentation, *2.3. Understanding a Bug*, rendered at
`https://bugzilla.readthedocs.io/en/latest/using/understanding.html` and
generated from
[`docs/en/rst/using/understanding.rst`](https://github.com/bugzilla/harmony/blob/main/docs/en/rst/using/understanding.rst)
in [`bugzilla/harmony`](https://github.com/bugzilla/harmony) (github.com).
Quoted from that `.rst` source.

Under *Importance (Priority and Severity):*

> "The Severity field indicates how severe the problem is—from blocker
> ("application unusable") to trivial ("minor cosmetic issue"). You
> can also use this field to indicate whether a bug is an enhancement
> request."

Under *Dependencies (Depends On and Blocks):*

> "If this bug cannot be fixed unless other bugs are fixed (depends
> on), or this bug stops other bugs being fixed (blocks), their
> numbers are recorded here."

Note that the source states this as a single sentence defining both
directions of one relationship, with "depends on" and "blocks" as
parenthetical labels — not as two separate field definitions.

Severity is a single-bug field; Depends On/Blocks is a relationship between
two bugs. They're independent — don't infer one from the other.
