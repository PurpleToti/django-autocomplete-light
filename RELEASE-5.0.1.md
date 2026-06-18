# 5.0.1 — patch release plan

Tracking: #1419 (BC break / customization hooks), #1420 (Django compatibility & upgrade path)

## Why this release exists

`5.0.0` is the correct home for the v4 architecture changes (it took the breaking
`4.0.1` content and bumped the major, adopting SemVer). What it still lacks is the
follow-through on the two open issues:

- **#1419** — downstream users had to monkey-patch internals (`Select2InitialRenderMixin`,
  the JS `url` getter, etc.). The new architecture should give them *clean, supported*
  override points so no monkey-patching is needed — and the one genuine bug behind the
  reports (`to_field_name` ignored on initial render) should be fixed.
- **#1420** — there is no per-release Django compatibility statement and no documented
  upgrade path.

`5.0.1` is a **patch**: it only ships bug fixes, additive hooks, and docs. No public
API is removed or changed in a breaking way (SemVer-safe).

This is the forward-looking counterpart to the `4.0.2` transition branch: `4.0.2`
keeps `>=4,<5` users working, `5.0.1` makes the v5 line the clean destination.

## Scope

### 1. Fix `to_field_name` on initial render (#1419, the real bug)

**Problem.** The widgets that filter pre-selected choices assume option values are
primary keys, even when the bound `ModelChoiceField` uses `to_field_name="uuid"` (or
any non-pk field). On edit forms the selected options therefore fail to render. This
is the root cause behind bheemaguli's report; their workaround only existed because
the library hardcodes `pk`.

Two hardcoded-`pk` paths to fix:

- `src/dal/widgets.py` — `QuerySetSelectMixin.filter_choices_to_render()` (~line 157):
  `queryset.filter(pk__in=[...])`.
- `src/dal_select2/widgets.py` — `ModelSelect2Multiple.filter_choices_to_render()`
  (~line 131): `When(pk=pk, ...)` + `queryset.filter(pk__in=pks)`.

**Fix.** Resolve the lookup field from the bound field instead of assuming `pk`:

```python
def _to_field_name(self):
    field = getattr(self.choices, 'field', None)
    return getattr(field, 'to_field_name', None) or 'pk'
```

Then build the lookups dynamically, e.g.:

```python
name = self._to_field_name()
values = [c for c in selected_choices if c]
qs = self.choices.queryset.filter(**{f'{name}__in': values})
# preserve Select2 order for the multiple case:
preserved = Case(*[When(**{name: v}, then=pos) for pos, v in enumerate(values)])
qs = qs.order_by(preserved)
```

Default behaviour (`to_field_name` unset → `pk`) is unchanged, so this is a bugfix,
not a BC break.

**Tests.** Add a `test_project` app (or extend an existing select2 app) with a model
keyed by `uuid` and a `ModelChoiceField(to_field_name="uuid")`, asserting that
pre-selected values render on an edit form. This is also the example jpic asked for in
the issue thread.

### 2. Document the supported customization hooks (#1419)

So downstream stops monkey-patching internals, document the *intended* override points
that already exist in 5.0.0:

- **Python:** `filter_choices_to_render()` as the single, supported place to control
  which initial choices render (now `to_field_name`-aware after item 1).
- **JS forward:** `window.AutocompleteLightBuildForward` — already the public hook the
  component calls per request; document it as the replacement for the old prototype
  patch of `AutocompleteSelectInput.url`.
- **Dark mode:** document that the JS `AutocompleteLightDarkMode` API was intentionally
  removed in favour of pure CSS (`:root[data-theme=...]` / `prefers-color-scheme`), and
  show the CSS-only way to force a theme — this is the migration note for anyone who
  called the old JS API.

Place these in `docs/` (e.g. a new `docs/customization.rst`, linked from `index.rst`).

### 3. Compatibility statement + upgrade guide (#1420)

- **CHANGELOG:** add an explicit `Compatibility: Python 3.11–3.14, Django 5.2–6.0`
  line to the `5.0.1` entry, and adopt this as a standing convention for future
  entries (the core ask of #1420).
- **Upgrade guide:** add `docs/upgrade.rst` (linked from `index.rst`) covering the
  v3/v4 → v5 path, and naming the **`4.0.2` transition release** as the Django
  stepping-stone:

  ```
  on Django 4.2:  dal old  →  dal 4.0.2 (supports Django 4.2 and 5.2)
  then:           bump Django 4.2 → 5.2
  then:           dal 4.0.2 → dal 5.0.1
  ```

  This directly answers #1420's "no single release spans 4.2↔5.2 LTS" complaint by
  pointing at `4.0.2`.

### 4. Version & changelog

- `pyproject.toml`: `version = "5.0.0"` → `"5.0.1"`.
- `CHANGELOG`: new `5.0.1` entry citing #1419 and #1420, with the compatibility line.

## Validation checklist

- [ ] new `to_field_name="uuid"` test app: selected options render on edit forms for
      both single and multiple select2 widgets.
- [ ] existing pk-based tests still pass (default path unchanged).
- [ ] `tox` green on `dj52` / `dj60`.
- [ ] docs build (`tox -e docs`) with the new customization + upgrade pages.
- [ ] no public symbol removed vs 5.0.0 (patch-safe).

## Out of scope

- Reverting any 5.0.0 architecture (that is the `4.0.2` branch's job).
- Re-adding the removed JS `AutocompleteLightDarkMode` API (CSS is the supported path;
  we document the migration instead).

## References

- Issues: #1419, #1420
- Bug locations: `src/dal/widgets.py` (`QuerySetSelectMixin.filter_choices_to_render`),
  `src/dal_select2/widgets.py` (`ModelSelect2Multiple.filter_choices_to_render`)
- Companion transition release: `4.0.2` (see `RELEASE-4.0.2.md` on the `4.0.2` branch)
