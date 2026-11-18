# Issue Resolution: Dataclass `default_factory` Field Treated as Required in Union Disambiguation

## 1. Upstream Issue Reference
- **Repository:** `python-attrs/cattrs`
- **Issue:** [#796](https://github.com/python-attrs/cattrs/issues/796) — `Dataclass default_factory field treated as required in union disambiguation`
- **Component:** `src/cattrs/disambiguators.py` (`create_default_dis_func`)

## 2. Problem Description
In `cattrs.disambiguators.create_default_dis_func`, union disambiguation identifies each member class by finding a unique attribute that has **no default** (i.e., a required field guaranteed to be present in input payloads).

When inspecting fields via `fields_dict(cl)`, dataclass fields are returned as `dataclasses.Field` objects. For dataclass fields defined with `default_factory` (e.g., `items: list[str] = field(default_factory=list)`), Python sets `.default` to `dataclasses.MISSING` and stores the factory callable in `.default_factory`.

Because `create_default_dis_func` only checked:
```python
if cl_fields[orig_name].default in (NOTHING, MISSING):
```
it mistakenly treated dataclass fields with `default_factory` as required attributes with no default. As a result:
- Structuring valid input payloads that omit the optional `default_factory` field failed with `ValueError: Couldn't disambiguate`.
- Truly ambiguous unions (where multiple dataclasses only had unique `default_factory` fields) were not rejected with `TypeError`.

## 3. Root Cause
`fields_dict(cl)` returns `dataclasses.Field` objects for dataclasses and `attrs.Attribute` objects for `attrs` classes. For a `dataclasses.Field`, `.default` is `MISSING` both when a field is required AND when `default_factory` is specified. Checking `.default` alone cannot distinguish a required field from a `default_factory` field.

## 4. Solution Approach
1. **Bridge the Dataclass vs Attrs Field Representation Gap:** `create_default_dis_func` uses `fields_dict(cl)` to find unique required (non-default) attributes. For `attrs` classes, factory defaults are stored in `field.default = Factory(...)`, so `field.default is NOTHING` is `False`. For `dataclasses`, however, `default_factory` leaves `field.default == MISSING` and stores the factory in `field.default_factory`.
2. **Check Both `default` and `default_factory`:** Instead of replacing `fields_dict` (which could impact performance or public helper expectations), extended the non-default predicate in `src/cattrs/disambiguators.py` to require BOTH `field.default in (NOTHING, MISSING)` AND `getattr(field, "default_factory", MISSING) is MISSING`. This keeps `attrs` behavior unchanged while giving `dataclasses` full parity.
3. **TDD & Regression Safety:** Added `test_dataclasses_default_factory` in `tests/test_disambiguators.py` testing both fallback disambiguation when `default_factory` fields are omitted and `TypeError` rejection for ambiguous unions.

## 5. Reproduction Test Case
Added `test_dataclasses_default_factory` in `tests/test_disambiguators.py`:
```python
def test_dataclasses_default_factory(converter):
    @dataclass
    class A:
        shared: int
        items: list[str] = dc_field(default_factory=list)

    @dataclass
    class B:
        shared: int
        b: int

    assert converter.structure({"shared": 1}, Union[A, B]) == A(shared=1, items=[])
    assert converter.structure({"shared": 1, "b": 2}, Union[A, B]) == B(shared=1, b=2)
```
Before the fix, `converter.structure({"shared": 1}, Union[A, B])` raised `ValueError: Couldn't disambiguate`.

## 6. Solution & Code Changes
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

## 7. Verification
- Reproduction tests (`test_dataclasses_default_factory`) pass across all 4 converter configurations (`BaseConverter`/`Converter`, `True`/`False`).
- Full disambiguator test suite passes with zero regressions.
- `ruff check src/cattrs/disambiguators.py tests/test_disambiguators.py` passes cleanly.
