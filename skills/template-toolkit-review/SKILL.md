---
name: template-toolkit-review
description: Use when reviewing or writing a Bugzilla 5.0+/Harmony Template Toolkit file (*.tmpl) - checking FILTER html discipline on interpolated data, correct use of the Hook.process extension points, or template cache invalidation after edits. Trigger for ".tmpl review", "does this template escape user data correctly", or "why isn't my template change showing up" (compiled template cache) questions.
---

# Bugzilla Template Toolkit Review

Bugzilla's templates are Template Toolkit (`.tmpl`) files under `template/<lang>/default/`,
gated by a real test (`t/004template.t`) in addition to convention. Every
rule below is sourced from that test, `Bugzilla::Hook`/`Bugzilla::Template`,
and bugzilla.readthedocs.io — see `reference/` for full quotes.

## `FILTER html` discipline

Any value interpolated into HTML output that isn't already known-safe
markup must go through the `html` filter. Per bugzilla.readthedocs.io's
Template Customisation docs: "you should take particular care about is the
need to properly HTML filter data that has been passed into the template.
If the data can possibly contain special HTML characters such as `<`, they
need to be converted to entity form. You use the `html` filter in the
Template Toolkit to do this." Concretely, the filter resolves to
`Bugzilla::Util::html_quote` (registered in `Bugzilla::Template`).

Review checklist for any `[% ... %]` interpolation into HTML context:
- Plain user-controlled value (summary, comment text, product name, etc.)
  → must carry `FILTER html`, e.g. `[% product.name FILTER html %]`.
- Value going into a `href`/`src`/URL context often needs a second layer —
  templates in the wild combine filters, e.g. `FILTER html_light FILTER js`
  for a value used inside both an HTML attribute and a JS string context.
  A single filter is not automatically wrong, but check *which* filter
  matches the actual output context, not just that one is present.
- `t/004template.t` maintains an explicit allow-list of filters that are
  considered defined/legitimate (`html_linebreak`, `js`, `base64`,
  `url_quote`, `css_class_quote`, `xml`, `quoteUrls`, `bug_link`, `csv`,
  `unitconvert`, `time`, `wrap_comment`, `none`, `ics`, `markdown`, plus
  `html`) — a filter outside that list is a red flag, not a stylistic
  choice.
- `t/004template.t` also forbids single-quoted `href='...'` attributes
  outright (tracked back to a real XSS, Bugzilla bug 926085) — hrefs must
  use double quotes so the `html`/`url_quote` filtering inside them is
  actually effective.

## Hook usage

Extensions inject template content via `Hook.process`, backed by
`Bugzilla::Hook`. The `template_before_process` hook itself fires on every
`$template->process` call and every in-template `PROCESS`/`INCLUDE`,
receiving `vars` (the full template variable set), `file` (the template
being processed), and `context` (the `Template::Context` object) — an
extension can use `file` to scope itself to one template rather than
touching every render.

Inline hook points look like:

```
[% Hook.process('custom_field', 'bug/create/create.html.tmpl') %]
...
[% Hook.process('after_custom_fields') %]
...
[% Hook.process("end") %]
```

`t/004template.t` requires `global/variables.none.tmpl` to have been
preprocessed for the `Hook` plugin to be available at all — a template that
uses `Hook.process` without going through the normal processing chain
(e.g. a hand-rolled `Template->process` call in a script) will fail with an
undefined `Hook` object, not a Bugzilla bug.

When reviewing a new hook point:
- Confirm the hook name is unique to the template/purpose — `Hook.process`
  dispatches by name, so a collision silently runs the wrong extension code.
- Comment convention in Bugzilla templates is `[%# comment text %]`,
  including for hook-point documentation, e.g. noting what an
  `after_custom_fields` hook is meant for.

## Template cache invalidation

Per bugzilla.readthedocs.io: `data/template` is where Template Toolkit
writes compiled (Perl) versions of the templates, functioning as `COMPILE_DIR`.
The docs are explicit: **"Do not directly edit the files in this
directory, or all your changes will be lost the next time Template Toolkit
recompiles the templates."** `Bugzilla::Template`'s `precompile_templates`
function ("compiles all of Bugzilla's templates in every language") is
invoked from `checksetup.pl` for exactly this purpose.

If a `.tmpl` edit isn't showing up:
- Check whether the compiled cache in `data/template` is stale rather than
  assuming the edit itself is wrong — a stale compile cache is a much more
  common cause of "my template change isn't showing up" than a logic bug.
- Never hand-edit anything under `data/template` — it's regenerated and
  will discard the edit.
- Re-running `checksetup.pl` (which calls `precompile_templates`) or
  clearing `data/template` forces a recompile.

See `reference/filters-and-hooks.md` and `reference/caching.md` for full
source quotes.
