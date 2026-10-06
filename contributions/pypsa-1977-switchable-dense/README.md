# PyPSA #1977 — Preserve `inds` Order and Handle Unknown Names in `get_switchable_as_dense`

## Project
- **Project:** PyPSA (Python for Power System Analysis)
- **Repository:** [https://github.com/PyPSA/PyPSA](https://github.com/PyPSA/PyPSA)
- **Issue:** [#1977 — "get_switchable_as_dense ignores order of inds and silently drops unknown names"](https://github.com/PyPSA/PyPSA/issues/1977)
- **Original PR:** [#1980 — "Fix get_switchable_as_dense ordering and unknown inds"](https://github.com/PyPSA/PyPSA/pull/1980)
- **Fork:** `VenkateshGN/PyPSA`
- **Branch:** `fix/1977-switchable-dense-order`
- **Commits:**
  - `d49428bc`: `FIX preserve inds order in get_switchable_as_dense`
  - `3f1d71ad`: `FIX handle inds order for stochastic networks`

---

## Issue Behavior

In PyPSA, networks allow querying time-series dynamic attributes for components (such as generators or lines) using `get_switchable_as_dense`. The function accepts an optional `inds` argument (an index or list of component names) to restrict the returned DataFrame columns.

The issue reported in [#1977](https://github.com/PyPSA/PyPSA/issues/1977) was that calling:

```python
n.get_switchable_as_dense(c, attr, inds=...)
```

did not properly respect the order of names supplied by `inds`. 

For example, given requested indices:
```python
inds = pd.Index(["g3", "g1", "missing"])
```

The existing behavior would:
- Return DataFrame columns in default component order (e.g., `["g1", "g3"]`) rather than requested order (`["g3", "g1"]`).
- Silently drop unknown or missing component names (such as `"missing"`) instead of raising an error.

### Desired Behavior
- The returned DataFrame columns should follow the exact order specified by the `inds` parameter.
- Requesting unknown component names should raise a `KeyError`, matching standard pandas `.loc`-style selection semantics.

### Minimal Reproduction Script

```python
import pandas as pd
import pypsa

n = pypsa.Network()
n.set_snapshots(range(2))
n.add("Bus", "b")
n.add(
    "Generator",
    ["g1", "g2", "g3"],
    bus="b",
    p_max_pu=[0.1, 0.2, 0.3],
)

# Calling get_switchable_as_dense with specific order and an unknown component
res = n.get_switchable_as_dense(
    "Generator",
    "p_max_pu",
    inds=pd.Index(["g3", "g1", "missing"])
)
```

### Conceptual Diagnosis
Conceptually, the existing implementation relied on intersecting indices using set operations or standard subset filtering that defaulted back to the network's underlying internal component order. Additionally, because filtering dropped non-matching keys without validation, missing names were silently ignored rather than causing an explicit failure.

---

## My Initial Investigation

Rather than modifying unrelated code or higher-level network descriptors immediately, I investigated the core array helper function where `get_switchable_as_dense` resolves dynamic series data.

The relevant implementation was in:
`pypsa/components/array.py` inside the `_as_dynamic()` method.

### Initial Approach
My initial change targeted direct column selection using standard pandas indexing:

```python
if inds is not None:
    res = res[inds]
```

### Why I Chose This Approach
- **Order Preservation:** Standard pandas DataFrame column indexing (`df[inds]`) naturally preserves the column order requested in `inds`.
- **Error Semantics:** If any key in `inds` does not exist in `res.columns`, pandas automatically raises a `KeyError`.
- **Cleanliness:** It directly matched desired API semantics without needing custom manual loops or explicit set intersections.

---

## Initial Regression Test & PR #1980

I added unit tests in `test/test_components_array.py` to cover both requirements:

```python
def test_get_switchable_as_dense_inds_order_and_unknown():
    # 1. Verifies requested order ["g3", "g1"] is preserved in output columns
    # 2. Verifies unknown names ["g3", "g1", "missing"] raise KeyError
    ...
```

### Local Validation
I executed local verification commands:

```bash
uv run pytest test/test_components_array.py -q
# Result: 16 passed, 1 warning

uv run ruff check test/test_components_array.py
# Result: All checks passed!

git diff --check
# Result: Passed cleanly
```

With local tests passing, I committed `d49428bc` and opened **PR #1980** against `master`. At this stage, the implementation was technically correct for standard non-stochastic networks.

---

## CI Failure & Technical Debugging

While the focused test `test_components_array.py` passed cleanly, the automated continuous integration pipeline failed on a stochastic network test.

### Failing Test
`test_get_switchable_inds_selects_names_in_all_scenarios` in `test/test_stochastic.py`.

### Failure Log / Traceback
```text
KeyError: "Index(['gen'], dtype='str') not in index"
```

Traceback path:
`pypsa/network/descriptors.py` → `get_switchable_as_dense` → `pypsa/components/array.py` → `_as_dynamic` → `res = res[inds]`

### Root Cause Analysis
My initial implementation assumed that `res.columns` is always a standard single-level pandas `Index` containing component names (e.g., `["g1", "g2", "g3"]`).

While that assumption holds true for standard networks, **stochastic networks** in PyPSA model multiple scenarios simultaneously. In stochastic networks, `res.columns` is represented as a **pandas `MultiIndex`** structured as:

```text
(scenario, component_name)
```

For example:
```python
MultiIndex([
    ("s1", "g1"),
    ("s1", "g2"),
    ("s2", "g1"),
    ("s2", "g2")
])
```

Executing `res[inds]` where `inds = ["gen"]` attempted a direct top-level column lookup for string `"gen"` against tuples `("s1", "gen")`. Because `"gen"` was not present at the top level of the `MultiIndex`, pandas raised a `KeyError`.

> **Note:** The CI failure was strictly caused by my initial implementation failing to handle `MultiIndex` column structures in stochastic networks. It was not a Python 3.14 issue, environment glitch, or flaky test.

---

## Second Investigation & Implementation

To support both standard and stochastic network representations, I updated `_as_dynamic()` in `pypsa/components/array.py` to inspect whether scenarios were present and handle the `MultiIndex` appropriately.

### Improved Implementation

```python
if inds is not None:
    inds_idx = pd.Index(inds)

    missing = inds_idx.difference(self.names)

    if len(missing) > 0:
        msg = f"Component names {missing.tolist()} not found in {self.name}"
        raise KeyError(msg)

    if self.has_scenarios:
        scenarios = res.columns.get_level_values("scenario").unique()
        cols = pd.MultiIndex.from_product(
            [scenarios, inds_idx], names=res.columns.names
        )
        res = res[cols]
    else:
        res = res[inds_idx]
```

### Key Technical Elements
1. **Normalizing `inds`:** Converting `inds` to `pd.Index` normalizes input iterables (lists, arrays, Index objects) and avoids tuple vs. list ambiguity.
2. **Explicit Validation against `self.names`:** Using `self.names` as the authoritative component-name collection explicitly detects missing components prior to column slicing, raising a clear `KeyError`.
3. **Standard Network Path:** For non-stochastic networks (`self.has_scenarios == False`), direct indexing `res[inds_idx]` is performed.
4. **Stochastic Network Path:** For stochastic networks, the code extracts unique scenario labels and constructs a full Cartesian product `pd.MultiIndex.from_product([scenarios, inds_idx])`. This preserves both scenario order and requested component order across all scenarios.

### Stochastic Regression Test

I added a new regression test in `test/test_stochastic.py`:

```python
def test_get_switchable_inds_preserves_order_and_rejects_unknown_names_stochastic():
    n = _two_bus_network()
    n.set_scenarios({"s1": 0.5, "s2": 0.5})
    n.add("Generator", "gen2", bus="bus1")

    inds = pd.Index(["gen2", "gen"])
    expected = [
        ("s1", "gen2"),
        ("s1", "gen"),
        ("s2", "gen2"),
        ("s2", "gen"),
    ]

    dense = n.get_switchable_as_dense(
        "Generator",
        "p_max_pu",
        inds=inds
    )

    assert dense.columns.tolist() == expected

    with pytest.raises(KeyError):
        n.get_switchable_as_dense(
            "Generator",
            "p_max_pu",
            inds=pd.Index(["gen", "missing"]),
        )
```

### Final Validation
```bash
uv run pytest test/test_components_array.py test/test_stochastic.py -q
# Result: 84 passed, 1 warning

uv run ruff check pypsa/components/array.py test/test_components_array.py test/test_stochastic.py
# Result: All checks passed!

git diff --check
# Result: Passed cleanly
```

I pushed commit `3f1d71ad` (`FIX handle inds order for stochastic networks`), which updated PR #1980. All CI checks passed cleanly.

---

## Important Maintainer Development & Strategy Shift

Following PR #1980, PyPSA collaborator **FabianHofmann** opened **PR #1982** titled:
`feat: warn when get_switchable_as_dense reorders or drops inds` (addressing issues [#1977](https://github.com/PyPSA/PyPSA/issues/1977) and [#1979](https://github.com/PyPSA/PyPSA/issues/1979)).

### Architectural & API Difference
- **My PR (#1980):** Implemented the desired target behavior immediately by raising `KeyError` on unknown names and reordering columns directly.
- **Maintainer Approach (#1982):** Proposed a backward-compatible migration strategy:
  - **Current Behavior (PyPSA 0.x / 1.x):** Keep existing returns, but emit a `FutureWarning` whenever `inds` would be reordered or dropped, or when unknown/duplicate names are supplied.
  - **Future Behavior (PyPSA 2.0):** Transition to strict `inds` ordering, preserve duplicates, and raise `KeyError` for unknown names.

### Context from Issue #1979 / Internal Callers
Investigation surrounding [#1979](https://github.com/PyPSA/PyPSA/issues/1979) and PR #1982 revealed that internal callers in `power_flow.py` and `clustering/spatial.py` were passing component names in ways that interacted with stochastic `MultiIndex` columns. Some callers passed `(scenario, name)` tuples rather than pure component names.

PR #1982 updated internal callers to explicitly use `.unique("name")` when querying stochastic networks, demonstrating that fixing the API required coordinating internal call sites alongside user-facing deprecations.

---

## Why PR #1980 Was Superseded

PR #1980 directly solved the core technical indexing problem. However, in mature open-source software libraries like PyPSA, changing API return types or exception behavior immediately can break downstream user scripts and dependent frameworks that relied on the old ordering or silent filtering.

The maintainers elected to adopt PR #1982's staged migration model:
1. Preserve existing behavior temporarily while emitting `FutureWarning`.
2. Give downstream users and projects notice before PyPSA 2.0.
3. Update internal call sites safely across the repository.

> **Key Takeaway:** My implementation was technically aligned with the eventual desired semantics, but the project chose a compatibility-first migration strategy.

---

## Final Outcome

- **PR #1980:** Superseded / Closed in favor of the maintainer deprecation strategy in PR #1982.
- **PR #1982:** Adopted by maintainers to manage the PyPSA 2.0 migration path.
- **Contributions:** Identified and reproduced the ordering and missing index bugs, created reproduction tests for both standard and stochastic network representations, and helped clarify the requirements for PyPSA's ultimate 2.0 API behavior.

---

## What I Learned

### Technical Concepts Learned
1. **Don't Assume Data Shape:** Column structures in pandas DataFrames can vary dynamically (e.g., standard `Index` vs. hierarchical `MultiIndex`).
2. **Understand the Full Call Path:** High-level API calls (`get_switchable_as_dense`) depend on internal descriptor helpers (`_as_dynamic`) and internal caller conventions.
3. **CI Is Part of Development:** A test suite passing locally for single-component scenarios does not guarantee safety across multi-scenario or stochastic modes.
4. **Multi-Representation Testing:** Unit and regression tests must explicitly cover both standard single-level indices and multi-level scenario `MultiIndex` configurations.
5. **API Correctness vs. Migration Strategy:** Immediate breaking fixes, even when conceptually superior, are often superseded by deprecation warnings (`FutureWarning`) in library development.
6. **Maintainer Collaboration:** Code reviews and alternative maintainer PRs are a normal part of open-source engineering; design decisions balance ideal semantics against backward compatibility.
7. **Pandas Indexing Nuances:** Difference between `.loc`-style selection, `Index.difference()`, and `MultiIndex.from_product()`.
8. **Read Before Changing:** Inspecting how PyPSA represents stochastic columns was essential to engineering a working solution for all network types.

### Engineering Lessons
- Reproduce bugs with minimal scripts before modifying source files.
- Keep initial changes targeted and readable.
- Treat CI failures as valuable diagnostic evidence rather than environment quirks.
- Trace upstream data representations when encountering `KeyError` in multi-dimensional indices.
- Evaluate backward-compatibility risks before introducing breaking API changes.
- Value project consistency and smooth user transitions over individual patch merge status.

---

## Interview Explanation

> "I contributed to PyPSA around issue #1977, where `get_switchable_as_dense` did not preserve the requested `inds` order and silently ignored unknown names. I first reproduced the behavior and implemented pandas-style selection to preserve order and raise `KeyError`. My focused tests passed, but CI exposed an important edge case in stochastic networks because their columns use a `MultiIndex` of scenario and component name. I investigated that failure, updated the implementation to handle both normal and stochastic networks, and added regression coverage. The tests then passed. Later, maintainers chose a compatibility-first approach in PR #1982, using `FutureWarning` before making the behavior breaking in PyPSA 2.0. That taught me that contributing to a mature project isn't only about making the code technically correct; you also have to consider API compatibility, downstream users, internal callers, and migration strategy."

---

## Links

- **PyPSA Repository:** [https://github.com/PyPSA/PyPSA](https://github.com/PyPSA/PyPSA)
- **Issue #1977:** [https://github.com/PyPSA/PyPSA/issues/1977](https://github.com/PyPSA/PyPSA/issues/1977)
- **PR #1980 (My Pull Request):** [https://github.com/PyPSA/PyPSA/pull/1980](https://github.com/PyPSA/PyPSA/pull/1980)
- **PR #1982 (Maintainer Migration PR):** [https://github.com/PyPSA/PyPSA/pull/1982](https://github.com/PyPSA/PyPSA/pull/1982)
- **Issue #1979:** [https://github.com/PyPSA/PyPSA/issues/1979](https://github.com/PyPSA/PyPSA/issues/1979)
