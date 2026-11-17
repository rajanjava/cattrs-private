# Issue Resolution: Dataclass default_factory field treated as required in union disambiguation

## Problem Description
In `cattrs.disambiguators.create_default_dis_func`, union disambiguation selects a unique attribute without a default to disambiguate union members.
When inspecting fields via `fields_dict(cl)`, dataclass fields are represented as `dataclasses.Field` objects. For dataclass fields defined with `default_factory` (e.g., `items: list[str] = field(default_factory=list)`), the `.default` attribute is set to `dataclasses.MISSING`, while the factory is stored in `.default_factory`.

Because `create_default_dis_func` checked:
```python
if cl_fields[orig_name].default in (NOTHING, MISSING):
```
it erroneously treated dataclass fields with a `default_factory` as required attributes having no default. Consequently, union disambiguation pinned classes to fields with `default_factory`, causing deserialization failures (`ValueError: Couldn't disambiguate`) when those fields were omitted, or failing to reject ambiguous unions (`TypeError`).

## Root Cause
`fields_dict(cl)` returns `dataclasses.Field` instances for dataclasses and `attrs.Attribute` instances for attrs classes.
For a `dataclasses.Field`, `field.default` is `MISSING` when either no default is provided or `default_factory` is used. A field only truly lacks a default if both `field.default in (NOTHING, MISSING)` and `getattr(field, "default_factory", MISSING) is MISSING`.

## Solution
Updated `src/cattrs/disambiguators.py`:
```python
field = cl_fields[orig_name]
if (
    field.default in (NOTHING, MISSING)
    and getattr(field, "default_factory", MISSING) is MISSING
):
    uniq_attrs_dict[maybe_renamed_attr_name] = cl
    made_progress = True
    break
```
Added tests in `tests/test_disambiguators.py` verifying that dataclasses with `default_factory` fields are correctly disambiguated when fields are omitted, and ambiguous unions raising `TypeError`.
