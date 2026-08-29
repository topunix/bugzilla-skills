# Template cache invalidation — sources

Source: Bugzilla documentation, *5.4. Templates* (Integrating with Bugzilla),
rendered at
`https://bugzilla.readthedocs.io/en/latest/integrating/templates.html` and
generated from
[`docs/en/rst/integrating/templates.rst`](https://github.com/bugzilla/harmony/blob/main/docs/en/rst/integrating/templates.rst)
in [`bugzilla/harmony`](https://github.com/bugzilla/harmony) (github.com).
Quotes below are taken from that `.rst` source; reStructuredText inline
markup (`:file:`, `:command:`, ``` `` ```) has been rendered as Markdown
code formatting, and the emphasis in *Do not* is the source's own.

On the compiled-template directory:

> "A directory `data/template` also exists; this is where Template Toolkit
> puts the compiled versions (i.e. Perl code) of the templates. *Do not*
> directly edit the files in this directory, or all your changes will be
> lost the next time Template Toolkit recompiles the templates."

On the remedy after editing templates:

> "You should run `./checksetup.pl` after editing any templates. Failure to
> do so may mean either that your changes are not picked up, or that the
> permissions on the edited files are wrong so the webserver can't read
> them."

Secondary source: [`Bugzilla/Template.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Template.pm)
in `bugzilla/harmony`, POD for `precompile_templates`:

> "Description: Compiles all of Bugzilla's templates in every language.
> Used mostly by `checksetup.pl`."

This is the underlying function `checksetup.pl` invokes to do the recompile
the docs instruct you to run.

## Practical implication for review/debugging

- **After editing any template, run `./checksetup.pl`.** This is the
  documented instruction, not just a workaround — treat "did you run
  checksetup.pl" as the first question when a template edit misbehaves.
- Template Toolkit compiles `.tmpl` sources to Perl and stores the result
  under `data/template`; a running Bugzilla serves from those compiled
  versions, not by re-parsing the `.tmpl` file on every request.
- **Two distinct documented failure modes** follow from skipping
  `checksetup.pl`, and they present very differently:
  - *Changes not picked up* — the edit is correct but the stale compiled
    version is still being served. Looks like "my change did nothing."
  - *Wrong file permissions* — the edited file can't be read by the
    webserver. Looks like a template error or a missing/blank page rather
    than a stale edit, so it's easy to misdiagnose as a syntax problem in
    the new code.
- When a `.tmpl` edit doesn't appear to take effect, check both of these
  before deep-diving the template logic itself.
- The compiled directory is derived output, not source: never hand-edit
  anything under `data/template` — the docs explicitly warn those changes
  are lost on the next recompile.

Note on naming: the compiled-template directory is `data/template`. The
cited page does not use the name `template_cache`, and does not mention
`COMPILE_DIR` — do not refer to either when citing this page.
