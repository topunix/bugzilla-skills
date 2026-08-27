# Dependency trees and duplicates — sources

## Dependency relations

Source: `Bugzilla::Bug::BUG_RELATIONS` in
[`Bugzilla/Bug.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Bug.pm)
(github.com/bugzilla/harmony):

```perl
use constant BUG_RELATIONS => (
  [dependencies => qw(dependson blocked)],
  [regressions  => qw(regressed_by regresses)],
);
```

Two independent relationship pairs exist in Harmony: the classic
Depends&nbsp;On/Blocks pair (`dependson`/`blocked`), and a separate
regression-tracking pair (`regressed_by`/`regresses`). A bug's dependency
tree and its regression chain are not the same graph — check which field
you're actually reading before drawing conclusions.

## Cycle prevention

`Bugzilla::Bug::_check_relationship_loop` walks a bug's existing dependency
tree recursively before a new edge is accepted:

```perl
sub _check_relationship_loop {
  # Generates a dependency tree for a given bug. Calls itself recursively
  # to generate sub-trees for the bug's dependencies.
  my ($field, $bug_id, $dep_id, $ids) = @_;
  return unless defined($dep_id);
  $ids = {} unless defined $ids;
  $ids->{$bug_id} = 1;
  if ($ids->{$dep_id}) {
    ThrowUserError("relationship_loop_single",
      {'bug_id' => $bug_id, 'dep_id' => $dep_id, ...});
  }
  ...
}
```

If bug A already (transitively) depends on bug B, Bugzilla refuses to let B
depend on A. Apply the same check manually when proposing a dependency edge
during triage, particularly across products/components where the existing
tree may not be visible in the bug you're looking at.

The dependency tree is also visible in the UI: bugzilla.readthedocs.io notes
that "clicking the Dependency tree link shows the dependency relationships
of the bug as a tree structure," with the ability to collapse, expand, and
filter resolved bugs from the view.

## Duplicate handling

Source: `Bugzilla::Bug` in
[`Bugzilla/Bug.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Bug.pm):

```perl
sub _check_dup_id {
  my ($self, $dupe_of) = @_;
  my $dupe_of_bug = $self->check($dupe_of, 'dup_id');
  # Make sure a loop isn't created when marking this bug as duplicate.
  _resolve_ultimate_dup_id($self->id, $dupe_of, 1);
  ...
}

sub set_dup_id {
  my ($self, $dup_id) = @_;
  # Make sure we have the DUPLICATE resolution
  if ($self->resolution ne 'DUPLICATE') {
    $self->set_bug_status(..., {resolution => 'DUPLICATE'});
  }
  my $dupe_of = new Bugzilla::Bug($self->dup_id);
  $dupe_of->add_comment("", {type => CMT_HAS_DUPE, extra_data => $self->id});
  ...
}
```

Key behaviors to rely on when triaging:
- `_check_dup_id` resolves the chain to the *ultimate* non-duplicate bug
  (`_resolve_ultimate_dup_id`) — marking bug C a duplicate of B, where B is
  already a duplicate of A, links C to A, not to B.
- Loop prevention mirrors the dependency-tree check: a bug can't be marked
  a duplicate of something that's (transitively) a duplicate of it.
- `set_dup_id` forces the resolution to `DUPLICATE` and files a
  `CMT_HAS_DUPE`-typed comment on the target — this is how a "has duplicates"
  comment thread gets built without a human writing "this bug now has a
  duplicate."

## Duplicate search on file

Bugzilla's report form (`enter_bug.cgi`) searches existing summaries in the
same product/component as a reporter fills in a summary, to surface likely
duplicates before the report is even filed. This is a fuzzy text match, not
a guaranteed catch — a duplicate with a differently-worded summary won't be
suggested, so triage should still check the dependency/duplicate graph by
hand for reports that read as familiar but weren't flagged automatically.
