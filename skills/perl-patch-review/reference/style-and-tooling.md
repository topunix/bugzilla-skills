# Style and gating checks — sources

All sourced from the public [`bugzilla/harmony`](https://github.com/bugzilla/harmony)
repository (github.com), current `main` branch, unless noted.

## `.perltidyrc`

[`.perltidyrc`](https://github.com/bugzilla/harmony/blob/main/.perltidyrc):

```
-pbp     # Start with Perl Best Practices
-w       # Show all warnings
-iob     # Ignore old breakpoints
-l=80    # 80 characters per line
-vmll
-ibc
-iscl
-hsc
-mbl=2   # No more than 2 blank lines
-i=2     # Indentation is 2 columns
-ci=2    # Continuation indentation is 2 columns
-vt=0    # Less vertical tightness
-pt=2    # High parenthesis tightness
-bt=2    # High brace tightness
-sbt=2   # High square bracket tightness
-wn      # Weld nested containers
-isbc    # Don't indent comments without leading space
```

## `.perlcriticrc`

[`.perlcriticrc`](https://github.com/bugzilla/harmony/blob/main/.perlcriticrc)
(abridged to the policy-relevant lines):

```
theme = freenode || core || certrec || certrule || performance || security
severity = 1

[InputOutput::RequireCheckedSyscalls]
severity = 2
functions = :builtins
exclude_functions = print say sleep binmode

[CodeLayout::RequireTrailingCommas]
severity = 2

[CodeLayout::ProhibitParensWithBuiltins]
severity = 2

[RegularExpressions::ProhibitUnusualDelimiters]
severity = 2

[Freenode::EmptyReturn]
severity = 2

# _build_* are allowed
[Subroutines::ProhibitUnusedPrivateSubroutines]
private_name_regex = _(?!_|build_)\w+

[Variables::RequireLocalizedPunctuationVars]
allow = @ARGV $ARGV %ENV %SIG

[Variables::ProhibitPunctuationVars]
allow = $@ $! $/ $^O $^V
```

The file also explicitly disables a long list of stock Perl::Critic
policies (`ProhibitPostfixControls`, `ProhibitMagicNumbers`,
`ProhibitPackageVars`, `RequireFinalReturn`, etc.) — don't flag a patch for
violating one of these unless it's still enabled in the actual
`.perlcriticrc` in the target checkout, since the disabled list is
intentional project policy, not an oversight.

## `.editorconfig`

[`.editorconfig`](https://github.com/bugzilla/harmony/blob/main/.editorconfig):
`root = true`; Unix (LF) line endings and a final newline required
everywhere; 2-space indentation for Perl files (`*.pl`, `*.PL`, `*.pm`,
`*.cgi`, `*.t`) and for templates/CSS/JS/YAML.

## `t/002goodperl.t`

[`t/002goodperl.t`](https://github.com/bugzilla/harmony/blob/main/t/002goodperl.t)
runs six checks per file:

1. Executable files must have shebang `#!/usr/bin/env perl` exactly;
   `.pm` module files must have no shebang.
2. Every file must have `use 5.14.0` (or pull it in transitively via
   Moo/Role::Tiny/Mojo).
3. `use strict` present (same Moo/Role::Tiny/Mojo exemption).
4. `use warnings` present (same exemption).
5. Every `Throw*Error(...)` call's message must be a localization tag, not
   an inline string with whitespace in it.
6. The pattern `=> $cgi->param(...)` (assigning a raw `$cgi->param` call as
   a hash value) is forbidden outright.

## `t/005whitespace.t`

[`t/005whitespace.t`](https://github.com/bugzilla/harmony/blob/main/t/005whitespace.t)
runs three checks per file:

```perl
if (grep /\t/, @contents) { fail("$file contains tabs --WARNING") }
if (grep /\r/, @contents) { fail("$file contains non-OS-conformant line endings --WARNING") }
if ($first_line =~ /\xef\xbb\xbf/) { fail("$file contains Byte Order Mark --WARNING") }
```

No tabs, no CR line endings, no UTF-8 BOM. It does **not** check trailing
whitespace at end-of-line — don't flag trailing whitespace as a hard gate
failure, only as a style nit if you choose to raise it.

## Developer's Guide (classic, still applicable)

Source: Bugzilla project [Developer's Guide](https://www.bugzilla.org/contributing/developer.html)
(bugzilla.org): the project's test suite historically checks "that all Perl
files have the appropriate shebang switches (-w plus -T where required),
and that `use strict` is present." Harmony's own `t/002goodperl.t` (above)
is the concrete, current implementation of that same intent for the
Harmony tree — where the two differ (e.g. Harmony's primary CGI entry
points no longer carry `-T` in their shebang), defer to what
`t/002goodperl.t` actually enforces today over the general Developer's
Guide language.

## `CONTRIBUTING.md`

Source: [`CONTRIBUTING.md`](https://github.com/bugzilla/harmony/blob/main/CONTRIBUTING.md)
(github.com/bugzilla/harmony): "Commits should be as small as possible,
while ensuring that each commit is correct independently" — each commit is
expected to compile and pass tests on its own, not just the branch tip.
Contributions are made as GitHub pull requests and reviewed by core
contributors before merging; the project follows the Mozilla Community
Participation Guidelines.
