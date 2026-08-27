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

Source: bugzilla.readthedocs.io / Bugzilla project documentation on closed
bug resolutions (consistent across the Bugzilla 5.x documentation set):

- **FIXED** — a fix has been checked in.
- **INVALID** — the problem described is not a bug.
- **WONTFIX** — the problem is a real bug, but will not be fixed.
- **DUPLICATE** — the problem is a duplicate of an existing bug report
  (requires a target bug id).
- **WORKSFORME** — the bug could not be reproduced with the information
  given.
- **MOVED** — the bug was moved to another bug tracker.

Older Bugzilla installs additionally shipped `LATER`, `REMIND`, and
`NOTABUG`/`NOTOURBUG`/`INCOMPLETE` variants as local customizations; these
are not guaranteed present on a default 5.0+/Harmony install and are not
asserted here as defaults.

## Severity vs. dependency "Blocks"

Source: bugzilla.readthedocs.io, *Using Bugzilla → Understanding a Bug*
(rendered at `https://bugzilla.readthedocs.io/en/latest/using/understanding.html`,
generated from `docs/en/rst/using/understanding.rst` in the
[`bugzilla/bugzilla`](https://github.com/bugzilla/bugzilla/blob/5.2/docs/en/rst/using/understanding.rst)
docs source that also feeds the Harmony docs):

> "The Severity field indicates how severe the problem is — from blocker
> ('application unusable') to trivial ('minor cosmetic issue'). You can
> also use this field to indicate whether a bug is an enhancement request."

> "Depends On ... this bug cannot be fixed unless other bugs are fixed ...
> Blocks ... this bug stops other bugs being fixed."

Severity is a single-bug field; Depends On/Blocks is a relationship between
two bugs. They're independent — don't infer one from the other.
