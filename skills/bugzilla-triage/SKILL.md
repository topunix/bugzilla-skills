---
name: bugzilla-triage
description: Use when triaging bugs in a Bugzilla 5.0+/Harmony installation - reading or building a bug's dependency (Depends On/Blocks) tree, deciding whether a new report is a duplicate, judging whether a bug is a "blocker" versus routine severity, or picking a legal status/resolution transition (e.g. moving a bug to CONFIRMED, RESOLVED, or VERIFIED). Covers the default status workflow, resolution meanings, dependency-loop rules, and how duplicates are recorded.
---

# Bugzilla Triage

Guidance for triaging bugs against the default Bugzilla 5.0+/Harmony data
model: status workflow, resolutions, dependency trees, and duplicates. Every
rule below is sourced from Bugzilla's public documentation or the public
`bugzilla/harmony` source tree — see the `reference/` files for full quotes
and links. If a convention isn't cited to one of those sources, treat it as
outside this skill's scope rather than assumed.

## Status and resolution

Bugzilla's default install workflow only permits specific status
transitions, enforced in code (`Bugzilla::Bug::_check_bug_status` throws
`illegal_bug_status_transition` for anything not in the graph) and defined
by `Bugzilla::Install::STATUS_WORKFLOW`:

```
(none) -> UNCONFIRMED | CONFIRMED | IN_PROGRESS
UNCONFIRMED -> CONFIRMED | IN_PROGRESS | RESOLVED
CONFIRMED   -> IN_PROGRESS | RESOLVED
IN_PROGRESS -> CONFIRMED | RESOLVED
RESOLVED    -> UNCONFIRMED | CONFIRMED | VERIFIED
VERIFIED    -> UNCONFIRMED | CONFIRMED
```

When triaging:
- Don't propose a transition outside this graph (e.g. `IN_PROGRESS` straight
  to `VERIFIED`) — it isn't legal in the default workflow.
- `RESOLVED` requires a resolution; moving off `RESOLVED`/`VERIFIED`
  back to an open status clears it (`_check_bug_status` in `Bugzilla::Bug`).
- A closed bug reopened to `UNCONFIRMED` also resets `everconfirmed`.
- `IN_PROGRESS` can require a target milestone if the installation has
  `musthavemilestoneonaccept` enabled — check before flagging a triage as
  incomplete over a missing milestone.

Resolution meanings when closing: `FIXED`, `INVALID` ("not a bug as
described"), `WONTFIX` ("valid bug, will not be fixed"), `DUPLICATE`,
`WORKSFORME`, `MOVED`. See `reference/status-and-resolution.md` for
definitions and sources.

## Dependency trees (Depends On / Blocks)

"Depends On" and "Blocks" are inverse views of the same relationship — a bug
that depends on others "cannot be fixed unless other bugs are fixed"; a bug
that blocks others "stops other bugs being fixed"
(bugzilla.readthedocs.io, *Understanding a Bug*). In code this is the
`dependencies` entry of `Bugzilla::Bug::BUG_RELATIONS` (`dependson` /
`blocked` fields); Harmony also tracks a separate `regressions` relation
(`regressed_by` / `regresses`) — don't conflate the two when reading a bug's
relationships.

Before adding a dependency edge, Bugzilla itself walks the existing tree
recursively (`_check_relationship_loop` in `Bugzilla::Bug`) and refuses to
create a cycle. When triaging a dependency tree by hand, apply the same
check: a bug cannot depend on something that (transitively) already depends
on it.

**"Blocker" is not the same field as "Blocks".** Severity `blocker` is a
per-bug field value meaning "application unusable"
(bugzilla.readthedocs.io, *Understanding a Bug*), independent of whether the
bug structurally blocks any other bug. A bug can be severity=blocker with an
empty Blocks list, and a bug with a long Blocks list can be severity=normal.
Judge "is this a blocker" on severity/impact; judge "what does this block"
on the dependency tree — don't use one to infer the other.

## Duplicate detection

Bugzilla surfaces likely duplicates at file time by searching existing
summaries in the same product/component as the reporter types (`enter_bug.cgi`).
When triaging a report that looks like a duplicate:
- Marking a bug DUPLICATE requires the target bug ID; `Bugzilla::Bug::_check_dup_id`
  resolves the *ultimate* duplicate target (so `dup_id` always points at a
  non-duplicate bug) and refuses to create a duplicate loop.
- `set_dup_id` forces the resolution to `DUPLICATE` if it isn't already, and
  adds a `CMT_HAS_DUPE`-typed comment to the target bug recording the link.
- Don't mark something a duplicate just because it's similar — confirm the
  underlying root cause matches, since the DUPLICATE resolution merges CC
  and vote weight onto the target bug.

See `reference/dependencies-and-duplicates.md` for source snippets.
