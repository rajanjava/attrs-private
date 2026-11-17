# Issue Resolution: Slots Declared on Attrs-Decorated Class Are Ignored / Lost

## 1. Upstream Metadata
- **Repository:** `python-attrs/attrs`
- **Issue Reference:** [#1637](https://github.com/python-attrs/attrs/issues/1637)
- **Title:** `slots declared on attrs-decorated class are ignored/lost`
- **Component:** `src/attr/_make.py` (`_ClassBuilder._create_slots_class`)

---

## 2. Problem Description & Root Cause
When an attrs class is decorated with `@attrs.define` (or `@attr.s(slots=True)`) and declares its own `__slots__` (for example, to maintain private instance variables that should not be managed as public attrs attributes):
```python
@attrs.define
class A:
    __slots__ = ("private",)
    public: str = "foo"
```
Any access or assignment to `a.private` on an instance failed with:
```
TypeError: descriptor 'private' for 'A' objects doesn't apply to a 'A' object
```

### Root Cause:
1. `_ClassBuilder._create_slots_class()` builds a new slotted class to replace the original decorated class.
2. When creating the class dictionary `cd` for the new class, `self._cls_dict` contains member descriptors created by Python for the original class `self._cls`.
3. `names` only contained `self._attr_names` (attrs attributes) and cached properties. It completely ignored `self._cls.__dict__.get("__slots__")`.
4. As a result, the newly generated class `cls = type(...)` had `cd["__slots__"]` containing only attrs attributes.
5. Meanwhile, `cd` retained the old descriptor pointing to the discarded `self._cls` type, causing Python to raise a `TypeError` when accessing the attribute on instances of the new type.

---

## 3. Reproduction Test
A test `test_slots_declared_on_class` was added to `tests/test_slots.py`:
```python
def test_slots_declared_on_class():
    """
    Slots declared on an attrs-decorated class are preserved and accessible.
    """
    @attrs.define
    class A:
        __slots__ = ("private",)
        public: str = "foo"

    a = A()
    a.private = 1
    assert a.public == "foo"
    assert a.private == 1
    assert "private" in A.__slots__
    assert not hasattr(a, "__dict__")
```
Prior to the fix, this test failed with:
```
TypeError: descriptor 'private' for 'A' objects doesn't apply to a 'A' object
```

---

## 4. Code Fix
In `src/attr/_make.py` within `_ClassBuilder._create_slots_class`:
1. Extracted and normalized any `own_slots` explicitly declared on `self._cls`.
2. Excluded `*own_slots` from `cd` so that old member descriptors from `self._cls` are not copied over to the new class dictionary.
3. Included `own_slots` in `names` (`self._attr_names + tuple(s for s in own_slots if s not in self._attr_names)`).
4. When `cls = type(...)` is called, Python creates new, valid slot descriptors for both the attrs attributes and the explicitly declared slots on the new type.

---

## 5. Verification Results
- **Reproduction Test:** `1 passed, 54 deselected in 0.14s` (`tests/test_slots.py -k test_slots_declared_on_class`).
- **Slots Test Suite:** `54 passed, 1 skipped in 0.27s` (`tests/test_slots.py`).
- **Core Test Suite:** `539 passed in 20.75s` (`tests/test_make.py`, `tests/test_funcs.py`, `tests/test_converters.py`, `tests/test_validators.py`).
- **Linter Check:** `ruff check` passed with zero errors.
- **Repository Commit:** Pushed cleanly to `rajanjava/attrs-private` on branch `main`.
