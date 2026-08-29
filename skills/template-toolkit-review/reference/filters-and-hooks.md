# FILTER discipline and hooks — sources

## `FILTER html`

Source: Bugzilla documentation, *5.4. Templates* (Integrating with Bugzilla),
rendered at
`https://bugzilla.readthedocs.io/en/latest/integrating/templates.html` and
generated from
[`docs/en/rst/integrating/templates.rst`](https://github.com/bugzilla/harmony/blob/main/docs/en/rst/integrating/templates.rst)
in [`bugzilla/harmony`](https://github.com/bugzilla/harmony) (github.com).
Quoted from that `.rst` source, with reStructuredText inline literals
(``` `` ```) rendered as Markdown code formatting:

> "One thing you should take particular care about is the need to properly
> HTML filter data that has been passed into the template. This means that
> if the data can possibly contain special HTML characters such as `<`, and
> the data was not intended to be HTML, they need to be converted to entity
> form, i.e. `&lt;`.  You use the `html` filter in the Template Toolkit to
> do this (or the `uri` filter to encode special characters in URLs).  If
> you forget, you may open up your installation to cross-site scripting
> attacks."

Two things in that sentence are easy to lose and both matter:

- The qualifier **"and the data was not intended to be HTML"** — the rule is
  not "filter everything," it's that data not meant to be markup must not be
  able to become markup. Content deliberately carrying HTML is the case
  `html_light` exists for.
- The parenthetical naming **`uri`** as the filter for URL contexts. HTML
  entity encoding is not URL encoding; `html` is the wrong tool for a value
  being interpolated into a URL.

Source: [`Bugzilla/Template.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Template.pm)
in `bugzilla/harmony` registers the `html` filter as:

```perl
html => \&Bugzilla::Util::html_quote,
```

## URL-context filters

The docs name `uri` for URL encoding. Scope note on what is and isn't
verified here:

- `uri` is a **stock Template Toolkit filter**, not one Bugzilla defines —
  it does not appear as a registered key in `Bugzilla/Template.pm`'s filter
  set, and the docs reference it as a Template Toolkit facility.
- `url_quote` is Bugzilla's own URL-encoding routine
  ([`Bugzilla::Util::url_quote`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Util.pm),
  percent-encoding everything outside `[a-zA-Z0-9_\-.]`), and appears in
  `t/004template.t`'s list of known-defined filters, so it is legitimate in
  templates.
- `css_class_quote` is documented in `Bugzilla/Template.pm` by an adjacent
  comment as "similar to url_quote but used a \ instead of a % as prefix.
  In addition it replaces a ' ' by a '_'." — it is for CSS class names, not
  a general URL filter.

In review, the question is whether the filter matches the **output context**,
not merely whether some filter is present: `html` for HTML text, a
URL-encoding filter for values placed into URLs, `js` for values landing in
JavaScript string context. A value interpolated into an `href` typically sits
in two contexts at once (URL inside an HTML attribute), which is why layered
filters appear in the real templates.

## Filter usage examples

Source: [`template/en/default/bug/create/create.html.tmpl`](https://github.com/bugzilla/harmony/blob/main/template/en/default/bug/create/create.html.tmpl)
(github.com/bugzilla/harmony):

```
value="[% product.name FILTER html %]"
<option value="[% comp.name FILTER html %]"
```

Combined/layered filters also appear in the template set, e.g.
`FILTER html_light FILTER js` where a value needs to be safe in both an
HTML attribute and a JS string context.

Comment convention, from the same file:

```
[%# Migration note: The following file corresponds to the old Param
  # 'entryheaderhtml'
  #%]

[%# Build the lists of assignees and QA contacts if "usemenuforusers" is enabled. %]
```

## `t/004template.t` gating checks

Source: [`t/004template.t`](https://github.com/bugzilla/harmony/blob/main/t/004template.t)
(github.com/bugzilla/harmony). Per-file checks:

1. **Referenced templates must exist** — anything the codebase `PROCESS`/
   `INCLUDE`s must exist in the English template tree or an extension's
   template directory (other languages may fall back to English).
2. **Syntax validity** — every template is parsed with `Template::Provider`
   before it's trusted.
3. **Filter allow-list** — filters used anywhere in the template set must
   be one of a known-defined set: `html`, `html_linebreak`, `js`, `base64`,
   `url_quote`, `css_class_quote`, `xml`, `quoteUrls`, `bug_link`, `csv`,
   `unitconvert`, `time`, `wrap_comment`, `none`, `ics`, `markdown`.
4. **Forbidden construct** — single-quoted `href='...'` attributes are
   blocked outright, tied back to Bugzilla bug 926085 (a security issue).
5. **`global/variables.none.tmpl` preprocessing** — templates depend on
   this being processed first to get plugins like `Hook` initialized;
   `t/004template.t` accounts for this in how it drives template parsing.

## Hook mechanism

Source: [`Bugzilla/Hook.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Hook.pm)
(github.com/bugzilla/harmony):

> "Bugzilla allows extension modules to drop in and add routines at
> arbitrary points in Bugzilla code."

- **Registration**: `finalize()` scans enabled extensions for lowercase
  method names (excluding anything matching `/^[A-Z_]+/`) and stores
  references to them in an internal `%HOOKS` table.
- **Invocation**: `Bugzilla::Hook::process("hookname", { arg => $value,
  ... })` runs every registered hook for that name, passing the extension
  object and the arguments hashref to each.
- **`template_before_process`**: fires "any time Bugzilla processes a
  template file, including calls to `$template->process`, `PROCESS`
  statements in templates, and `INCLUDE` statements in templates." Receives
  `vars` (full template variable set), `file` (the template filename), and
  `context` (a `Template::Context` object) — an extension can inspect
  `file` to scope itself to a specific template.

Inline `Hook.process` calls in templates (from
`template/en/default/bug/create/create.html.tmpl`):

```
[% Hook.process('custom_field', 'bug/create/create.html.tmpl') %]
[% Hook.process('after_custom_fields') %]
[% Hook.process("end") %]
```
