# Template cache invalidation — sources

Source: bugzilla.readthedocs.io, *Integrating with Bugzilla → Templates*
(`https://bugzilla.readthedocs.io/en/latest/integrating/templates.html`):

> "The `template_cache` directory is used by Bugzilla as `COMPILE_DIR` for
> Template Toolkit." A directory `data/template` holds the compiled (Perl
> code) versions of the templates that Template Toolkit produces. "Do not
> directly edit the files in this directory, or all your changes will be
> lost the next time Template Toolkit recompiles the templates."

Source: [`Bugzilla/Template.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Template.pm)
in [`bugzilla/harmony`](https://github.com/bugzilla/harmony) (github.com),
POD for `precompile_templates`:

> "Description: Compiles all of Bugzilla's templates in every language.
> Used mostly by `checksetup.pl`."

## Practical implication for review/debugging

- Template Toolkit compiles `.tmpl` sources to Perl and caches the result
  under `data/template` (`COMPILE_DIR`); a running Bugzilla process serves
  from that compiled cache, not by re-parsing the `.tmpl` file on every
  request.
- A `.tmpl` edit that doesn't appear to take effect is, per this
  documented behavior, more likely a stale compiled cache than a logic
  error in the new template code — check cache freshness before deep-diving
  the template logic itself.
- The compiled cache directory is derived output, not source: never hand-
  edit anything under `data/template` — the docs explicitly warn changes
  there are lost on the next recompile.
- `precompile_templates` (invoked from `checksetup.pl`) is the documented,
  intentional way to force a full recompile across every configured
  language.
