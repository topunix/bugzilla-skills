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

Any value interpolated into output that isn't already known-safe markup must
go through a filter matching its output context. Per the Bugzilla docs,
*5.4. Templates*:

> "One thing you should take particular care about is the need to properly
> HTML filter data that has been passed into the template. This means that
> if the data can possibly contain special HTML characters such as `<`, and
> the data was not intended to be HTML, they need to be converted to entity
> form, i.e. `&lt;`.  You use the `html` filter in the Template Toolkit to
> do this (or the `uri` filter to encode special characters in URLs).  If
> you forget, you may open up your installation to cross-site scripting
> attacks."

Concretely, `html` resolves to `Bugzilla::Util::html_quote` (registered in
`Bugzilla::Template`). Note the two filters the docs name for two different
contexts: **`html` for HTML text, `uri` for URLs** — entity encoding is not
URL encoding, and using the first where the second belongs leaves the value
wrong and potentially unsafe.

Review checklist for any `[% ... %]` interpolation:
- Plain user-controlled value (summary, comment text, product name, etc.)
  into HTML text → must carry `FILTER html`, e.g.
  `[% product.name FILTER html %]`.
- Value going into a `href`/`src`/URL context → needs URL encoding, not just
  HTML escaping. The docs name `uri` (a Template Toolkit built-in);
  Bugzilla additionally ships `url_quote`
  (`Bugzilla::Util::url_quote`), which is in `t/004template.t`'s
  known-filter list.
- A value inside a URL *inside* an HTML attribute is in two contexts at
  once, which is why templates in the wild layer filters, e.g.
  `FILTER html_light FILTER js` for a value spanning HTML and JS string
  context. A single filter is not automatically wrong — check *which*
  filter matches the actual output context, not just that one is present.
- Note the docs' qualifier "and the data was not intended to be HTML": the
  rule targets data that must not be able to become markup. Content
  deliberately carrying HTML is what `html_light` is for.
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

The docs give a direct instruction: **"You should run `./checksetup.pl`
after editing any templates. Failure to do so may mean either that your
changes are not picked up, or that the permissions on the edited files are
wrong so the webserver can't read them."** `Bugzilla::Template`'s
`precompile_templates` ("compiles all of Bugzilla's templates in every
language") is the function `checksetup.pl` calls to do it.

Compiled templates live in `data/template`, and the docs warn: **"*Do not*
directly edit the files in this directory, or all your changes will be lost
the next time Template Toolkit recompiles the templates."**

If a `.tmpl` edit isn't showing up, the docs describe **two** failure modes
from skipping `checksetup.pl`, and they look different:
- **Changes not picked up** — the edit is fine, but the stale compiled
  version is still being served. Presents as "my change did nothing."
- **Wrong file permissions** — the webserver can't read the edited file.
  Presents as an error or blank output, so it's easily misread as a syntax
  problem in the new template code.

Check both before concluding the template logic is wrong. Never hand-edit
anything under `data/template` — it's regenerated and the edit is discarded.

See `reference/filters-and-hooks.md` and `reference/caching.md` for full
source quotes.
