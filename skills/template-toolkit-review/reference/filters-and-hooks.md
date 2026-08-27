# FILTER discipline and hooks — sources

## `FILTER html`

Source: bugzilla.readthedocs.io, *Integrating with Bugzilla → Templates*
(rendered at `https://bugzilla.readthedocs.io/en/latest/integrating/templates.html`,
also present in the Harmony-specific docs at
`https://bugzilla.readthedocs.io/projects/harmony/en/latest/integrating/templates.html`):

> "One thing you should take particular care about is the need to properly
> HTML filter data that has been passed into the template. If the data can
> possibly contain special HTML characters such as `<`, they need to be
> converted to entity form. You use the `html` filter in the Template
> Toolkit to do this."

Source: [`Bugzilla/Template.pm`](https://github.com/bugzilla/harmony/blob/main/Bugzilla/Template.pm)
in [`bugzilla/harmony`](https://github.com/bugzilla/harmony) (github.com)
registers the filter as:

```perl
html => \&Bugzilla::Util::html_quote,
```

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
