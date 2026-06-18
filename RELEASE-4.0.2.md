# 4.0.2 — transition release plan

Tracking: #1419 (BC break / SemVer), #1420 (Django compatibility & upgrade path)

## Why this release exists

The `dal_alight` rewrite and the widget/view/JS API churn that came with it were
backward-incompatible, but they shipped inside the **4.0.0** release without a major
version bump. Under SemVer those changes belong in their own major (5.0.0), not in a
4.x line. Concretely:

- `4.0.0` and `4.0.1` are essentially identical source — `4.0.1` only bumped the
  version and added a changelog line for #1417. The break was already in `4.0.0`.
- `4.0.1` was then **deleted from PyPI** and re-published as `5.0.0`. Deleting a
  published version broke every install pinned to `>=4,<5` (see #1419, Gagaro's
  deployments).

`5.0.0` already carries the breaking changes, which is correct. What is missing is a
**working, backward-compatible 4.x release** so that:

1. installs pinned to `>=4,<5` resolve again, and
2. there is a clean stepping-stone for the Django upgrade path requested in #1420.

`4.0.2` is that release: it **reverts the SemVer-breaking changes back out of the 4.x
line** (they stay in 5.0.0) and adds **Django 4.2 LTS support** so it can act as a
transition version.

### Intended upgrade path (the #1420 ask)

```
dal 4.0.1/old  ──▶  dal 4.0.2 (Django 4.2)  ──▶  bump Django 4.2 → 5.2  ──▶  dal 5.0.0
                    backward-compatible hooks       (4.0.2 supports both)     new web component
```

## Scope

### 1. Revert the SemVer-breaking refactor (it belongs only in 5.0.0)

Revert the public-API changes that broke downstream customizations, restoring the
4.x extension surface people built on:

- **Revert `0c2950f0`** ("refactor: address jpic review — widgets, views, JS, CSS cleanup"):
  - restore `WidgetMixin.render()` component-wrapping and
    `AlightWidgetMixin.component = 'autocomplete-select'` (the class attribute the
    generic render path reads via the MRO);
  - restore `BaseQuerySetView.post()` returning JSON directly (instead of the
    `_post()` split + per-frontend `post()`), so subclasses overriding `post()` work;
  - restore the JS `AutocompleteSelectInput.url` **prototype-patch** forward
    mechanism (instead of the `window.AutocompleteLightBuildForward` hook), which is
    the documented JS extension point;
  - restore `window.AutocompleteLightDarkMode` (`.initialize()` / `.toggle()`).
- **Revert `12efd270`** ("consolidate generic autocomplete code into dal core"):
  - restore `Select2InitialRenderMixin` — the importable MRO hook that
    `to_field_name` / UUID workarounds subclass (bheemaguli's case in #1419).

**Do NOT revert `9eabac0c`** ("add Select2InitialRenderMixin"): it *adds* the mixin.
Reverting `12efd270` already brings the mixin back; touching `9eabac0c` would remove
it again.

Net effect: the v4 customization hooks are restored on the 4.x line, while 5.0.0
keeps the new, cleaner architecture.

> Open item: a true fix for the UUID/`to_field_name` case (so users don't have to
> monkey-patch `Select2InitialRenderMixin` at all) is tracked for **5.1** — make the
> initial-render path honour `to_field_name` instead of assuming `pk`. Out of scope
> for this transition release.

### 2. Restore the importable packages dropped by `3a615b60`

`3a615b60` ("Remove residual legacy code and dead packages") deleted three
distribution packages. Removing importable packages is a BC break for any `>=4,<5`
user who imported them, so two of the three come back on the 4.x line:

| Package | Was for | Action |
| --- | --- | --- |
| `dal_genericm2m_queryset_sequence` | bridge for `django-generic-m2m` | **Restore** (small `fields.py`, cheap, niche but breaking to drop) |
| `dal_gm2m_queryset_sequence` | bridge for `django-gm2m` | **Restore** (same) |
| `dal_legacy_static` | vendored Select2 static for *old* Django admin | **Leave removed** — Django 4.2 admin already bundles Select2, so it is genuinely dead even for the transition target |

Restore the two `*_queryset_sequence` packages from their pre-`3a615b60` state and
re-add them to the packaging (`pyproject.toml` package discovery) so they ship again.

### 3. QoL: support Django 4.2 LTS (the "transition" part)

**Feasibility:** 4.0.0's headline break — dropping Python <3.11 / Django <5.2 — was
done by *policy* (packaging + tox + CI), not by code. A scan of `src/` found **no
Django-5.2-only APIs** (no `GeneratedField`, `db_default`, etc.), so re-supporting
Django 4.2 is a configuration change rather than a revert, and looks code-feasible —
to be confirmed by an actual `tox -e dj42` run.

Allow `4.0.2` to run on both Django 4.2 and 5.2 so users can upgrade dal first, then
Django:

- **`pyproject.toml`**
  - `dependencies = ["django>=4.2"]`
  - add classifiers `Framework :: Django :: 4.2`, `:: 5.0`, `:: 5.1` (alongside 5.2).
  - `requires-python`: must satisfy both Django 4.2 (Python ≤3.12) and 5.2
    (Python ≥3.10) → propose `>=3.10,<3.13`.
    ⚠ Verify the `src/` codebase has no 3.13+-only syntax before lowering the floor
    from the current `>=3.11`.
- **`tox.ini`**
  - add a `dj42` factor, e.g. `py{310,311,312}-dj42`, with `dj42: Django==4.2.*`.
  - keep the existing `dj52` / `dj60` envs.
- **`.github/workflows/ci.yml`** — add the 4.2 matrix entries.

### 4. Version & changelog

- `pyproject.toml`: set `version = "4.0.2"` on this branch (it is currently `5.0.0`,
  inherited from master).
- `CHANGELOG`: add a `4.0.2` entry, citing #1419 and #1420, and stating the supported
  matrix explicitly: **Python 3.10–3.12, Django 4.2–5.2** (addresses #1420's request
  for per-release compatibility documentation).

## Validation checklist

- [ ] `tox` green across `dj42` and `dj52`.
- [ ] `from dal_select2.widgets import Select2InitialRenderMixin` resolves again
      (bheemaguli's import).
- [ ] forward data still appended to autocomplete URLs (restored prototype patch).
- [ ] Django admin add/change-related popup still updates the deck.
- [ ] dark-mode JS API present again for downstreams that call it.
- [ ] `dal_genericm2m_queryset_sequence` and `dal_gm2m_queryset_sequence` importable
      again and shipped in the wheel/sdist.
- [ ] a fresh install pinned `django-autocomplete-light>=4,<5` resolves to 4.0.2.

## References

- Issues: #1419, #1420
- Breaking commits reverted here: `0c2950f0`, `12efd270`
- Packages restored here (dropped by `3a615b60`): `dal_genericm2m_queryset_sequence`,
  `dal_gm2m_queryset_sequence` (`dal_legacy_static` intentionally left out)
- Counterpart major release that keeps these changes: `5.0.0` (`5fcbce80`)
