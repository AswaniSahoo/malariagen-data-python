# PR #969 Analysis: Adding "Lower Triangle" Annotation to `plot_pairwise_average_fst`

**PR:** [#969](https://github.com/malariagen/malariagen-data-python/pull/969)
**Issue:** [#820](https://github.com/malariagen/malariagen-data-python/issues/820)
**Author:** Aswani Sahoo ([@AswaniSahoo](https://github.com/AswaniSahoo))
**Reviewer/Merger:** Jon Brenas ([@jonbrenas](https://github.com/jonbrenas))
**Merged:** 2026-02-28
**Diff:** +15 lines, −6 lines across 3 files

---

## 1. What the PR Does

This PR adds `"lower triangle"` as a third option to the existing `annotation` parameter in `plot_pairwise_average_fst()`. When selected, only the lower triangle of the pairwise Fst heatmap is displayed — the upper triangle is left blank. This is a common visual convention in symmetric pairwise matrices, where the upper and lower triangles are mirror images; showing both halves adds visual clutter without extra information.

Before this change, the `annotation` parameter accepted two values:
- `"standard error"` — fill the upper triangle with SE values
- `"Z score"` — fill the upper triangle with Z-score values (Fst/SE)
- `None` (default) — fill the upper triangle with the same Fst values as the lower triangle

After this change, there is a fourth behaviour:
- `"lower triangle"` — leave the upper triangle empty (rendered as blank cells)

---

## 2. Files Changed

| File | Changes | Purpose |
|------|---------|---------|
| [`malariagen_data/anoph/fst_params.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/malariagen_data/anoph/fst_params.py#L37-L46) | +6 −4 | Extended the `Literal` type and updated the docstring |
| [`malariagen_data/anoph/fst.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/malariagen_data/anoph/fst.py#L512-L595) | +5 −2 | Handled the new case at three distinct points in `plot_pairwise_average_fst()` |
| [`tests/anoph/test_fst.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/tests/anoph/test_fst.py#L257-L264) | +4 −0 | Added a test assertion inside the existing `check_pairwise_average_fst` helper |

---

## 3. Key Design Choice: Extending `Literal` vs. Adding a Boolean Flag

The central design decision was **how to expose the new option to the user**. There were two realistic approaches:

### Option A (chosen): Extend the existing `annotation` Literal type

```python
# Before
annotation: TypeAlias = Annotated[
    Optional[Literal["standard error", "Z score"]],
    ...
]

# After
annotation: TypeAlias = Annotated[
    Optional[Literal["standard error", "Z score", "lower triangle"]],
    ...
]
```

### Option B (rejected): Add a separate boolean parameter

```python
def plot_pairwise_average_fst(
    self,
    fst_df,
    annotation = None,
    show_lower_only: bool = False,   # new parameter
    ...
)
```

### Why Option A was the right choice

**a) The `annotation` parameter already controls upper-triangle behaviour.** Looking at the function's logic, `annotation` determines what data goes into `fig_df.loc[cohort1, cohort2]` — the upper triangle cells. Adding a boolean would create a second parameter that also controls upper-triangle behaviour, which introduces ambiguity. What should happen when a user passes `annotation="standard error", show_lower_only=True`? That combination is contradictory — you cannot simultaneously fill the upper triangle with SE values AND leave it empty. Option A avoids this conflict entirely because the user can only choose one behaviour.

**b) The `@_check_types` decorator provides free runtime validation.** The codebase uses a custom `_check_types` decorator (defined in [`util.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/malariagen_data/util.py#L1141-L1177)) that inspects `typing.get_type_hints()` on each function call and runs `typeguard.check_type()` against the resolved types. Because the parameter types are defined using `Annotated[Optional[Literal[...]]]` aliases in `fst_params.py`, adding `"lower triangle"` to the `Literal` means the decorator will automatically accept it as valid input — and automatically reject any invalid string — without writing any extra validation code. A boolean parameter would work too, but it adds a new parameter to validate rather than extending an existing validated set.

**c) The `Annotated` wrapper carries the docstring.** The `numpydoc_decorator`'s `@doc` decorator reads docstrings from the `Annotated` metadata. By updating the `Annotated` wrapper's docstring in `fst_params.py`, the new `"lower triangle"` option is documented automatically. A separate boolean would require adding a separate docstring entry in the `@doc(parameters=dict(...))` call — more places to maintain.

**d) The API surface stays minimal.** Issue [#366](https://github.com/malariagen/malariagen-data-python/issues/366) shows an ongoing effort to keep the API manageable as the codebase grows. Adding parameters has a real cost — every new parameter appears in help text, autocompletion, and documentation. Extending an existing parameter's value set avoids this.

---

## 4. The Three Code Changes in `fst.py` — What Each Does and Why

Understanding the implementation required tracing through the full `plot_pairwise_average_fst` method to see how the `annotation` parameter affects three separate stages of plot construction.

### 4a. Title text exclusion (line 540)

```python
# Before:
if annotation is not None:
    title += " ⧅ " + annotation

# After:
if annotation is not None and annotation != "lower triangle":
    title += " ⧅ " + annotation
```

**What the existing code does:** When annotation is `"standard error"` or `"Z score"`, the title becomes `"Fst ⧅ standard error"` or `"Fst ⧅ Z score"`, telling the user that the upper triangle contains different data than the lower triangle.

**Why `"lower triangle"` is excluded:** The strings `"standard error"` and `"Z score"` describe what *data* is in the upper triangle — they are content descriptors. `"Lower triangle"` describes a *display mode* — there is no additional data to label. A title like `"Fst ⧅ lower triangle"` would be confusing because it implies there is something in the upper triangle called "lower triangle," which is the opposite of what the mode does.

### 4b. The `pass` statement in the data-filling loop (lines 553-555)

```python
elif annotation == "lower triangle":
    # Leave the upper triangle as NaN (empty).
    pass
```

**Context:** The existing loop iterates over every pair `(cohort1, cohort2, fst, se)` from the Fst dataframe. For each pair, it always fills `fig_df.loc[cohort2, cohort1] = fst` (the lower triangle). Then it decides what to put in the *upper* triangle cell `fig_df.loc[cohort1, cohort2]` based on `annotation`:

- `None` → fill with the same `fst` value (symmetric heatmap)
- `"standard error"` → fill with `se`
- `"Z score"` → fill with `fst / se` (with a guard: if `se == 0`, writes `np.nan` instead to avoid division by zero)
- `"lower triangle"` → **do nothing** (`pass`)

**Why `pass` works:** The `fig_df` DataFrame was initialised from `pd.DataFrame(columns=cohorts, index=cohorts)`, which creates a DataFrame full of `NaN` values. By simply not writing to the upper-triangle cells, they remain `NaN`. When Plotly's `px.imshow()` renders the DataFrame, `NaN` cells are displayed as blank (the default `white` background shows through) with no text annotation. This means zero additional code is needed to create the visual effect — the `pass` statement is the entire implementation.

**Note:** The `"Z score"` branch already uses `NaN` as a blank-cell mechanism (for the `se == 0` edge case). This confirms that the `pass` approach for `"lower triangle"` is not an improvisation — it reuses a pattern already established by the existing code.

**Why not set the value explicitly to `np.nan`?** That would also work (`fig_df.loc[cohort1, cohort2] = np.nan`), but it is redundant — the cells are already `NaN` by default. The `pass` with a descriptive comment makes the intent clearer: the code is deliberately choosing *not* to assign a value, rather than assigning a missing value.

### 4c. The `zmax` colour scaling guard (line 561)

```python
# Before:
if annotation is not None and zmax is None:
    zmax = 1e9

# After:
if annotation is not None and annotation != "lower triangle" and zmax is None:
    zmax = 1e9
```

**What the existing code does:** When the upper triangle contains SE or Z-score values, they can have very different magnitudes from the Fst values in the lower triangle (e.g., Fst ranges 0–1, but Z-scores can be 10+). If both share the same colour scale, the Fst values would all appear as one colour. Setting `zmax = 1e9` effectively neutralises the colour scale so that the heatmap is not misleadingly coloured by the upper triangle's different value range.

**Why `"lower triangle"` is excluded:** When the upper triangle is empty (NaN), there is no conflicting data. The colour scale should map normally to the Fst values in the lower triangle (0 to the maximum observed Fst). If `zmax` were set to `1e9`, all Fst cells would appear as nearly the same pale colour because they would all be near zero relative to one billion — defeating the purpose of the colour gradient.

---

## 5. The `annotation` Parameter's Dual Role — The Obstacle

The biggest obstacle in implementing this change was recognising that `annotation` serves two distinct purposes simultaneously:

1. **Data role:** It determines what numerical values fill the upper triangle cells (Fst, SE, Z-score, or nothing)
2. **Presentation role:** It controls the plot title text and colour scaling behaviour

The existing two values (`"standard error"` and `"Z score"`) happen to align both roles cleanly — they describe both what data goes in the upper triangle AND what should appear in the title. But `"lower triangle"` only makes sense as a presentation choice (don't show the upper half) — it doesn't describe data content. This meant adding `annotation != "lower triangle"` guards at both presentation-level code points (title and `zmax`) while only needing a `pass` at the data-level code point.

This dual role also explains why the condition check is `annotation != "lower triangle"` rather than a more general approach. The new option is semantically different from the existing ones — it is the only `annotation` value that removes data rather than replacing it — so it requires explicit exclusion from the presentation logic that was written for the "replace" cases.

---

## 6. How the Type System Enabled a Minimal Change

The codebase's type annotation infrastructure made this change unusually clean:

1. **Type definition** in `fst_params.py` — single line change to the `Literal` type
2. **Runtime validation** via `@_check_types` — automatic, no code needed
3. **Documentation** via `Annotated` wrapper — updated in the same file as the type
4. **API consistency** — the function signature is unchanged; no new parameter

The total implementation is 3 lines of actual logic (`pass`, and two `!= "lower triangle"` guards), plus the type definition change and docstring update. This is possible because the codebase was designed with the `Annotated[Optional[Literal[...]]]` pattern specifically to make parameter extensions minimal — the type, validation, and documentation all live in one place (`fst_params.py`), and the `@_check_types` + `@doc` decorators propagate the change automatically.

---

## 7. Testing

The test was added inside the existing `check_pairwise_average_fst` helper function in [`test_fst.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/tests/anoph/test_fst.py#L213-L264), which is called by multiple parametrised test cases (`test_pairwise_average_fst_with_str_cohorts`, `test_pairwise_average_fst_with_min_cohort_size`, `test_pairwise_average_fst_with_dict_cohorts`, `test_pairwise_average_fst_with_sample_query`). This means the `"lower triangle"` option is tested across multiple cohort configurations (by country, by admin1_year, by admin2_month, by custom dict) and against **three** species simulators via `@parametrize_with_cases`: Ag3 (*Anopheles gambiae* complex), Af1 (*Anopheles funestus*), and Adir1 (*Anopheles dirus*).

The test asserts that `api.plot_pairwise_average_fst(fst_df, annotation="lower triangle", show=False)` returns a valid `go.Figure` instance, following the same pattern used for the existing `"standard error"` and `"Z score"` tests immediately above it.

**What the test does not check:** It verifies the function runs without error and returns the right type, but does not inspect the figure contents (e.g., verifying that specific cells are actually NaN). This is consistent with how the existing annotation options are tested — the test suite treats the plotting code as a smoke test, confirming it produces a figure, while relying on the data logic tests (DataFrame shape, column names, value ranges) for correctness guarantees on the underlying data.

**One nearby codebase inconsistency worth noting (not introduced by this PR):** The `@doc()` call on `plot_pairwise_average_fst` documents a `parameters=dict(annotate_se=...)` entry, but the function signature has no `annotate_se` parameter — the function uses `annotation` instead. This is a leftover from an earlier version of the API. It does not affect runtime behaviour (the `@doc` decorator only affects the docstring), but it means the generated documentation mentions a parameter that does not exist. Discovering this while tracing through the three code changes in Section 4 was a good reminder that the areas around a PR change can contain pre-existing issues.

---

## 8. Broader Observations from Working on This PR

**The cooperative multiple inheritance pattern:** `AnophelesFstAnalysis` inherits from `AnophelesSnpData` and uses `super().__init__(**kwargs)` with the comment *"this class is designed to work cooperatively"*. This is part of the ongoing refactoring described in [issue #366](https://github.com/malariagen/malariagen-data-python/issues/366), where the large `AnophelesDataResource` class is being broken into focused mixins. Understanding this pattern was necessary because the Fst methods call `self.snp_allele_counts()` and `self.sample_metadata()` which are defined in sibling mixin classes — they are resolved through Python's MRO at runtime, not through direct inheritance.

**The `Annotated` + `TypeAlias` pattern as machine-readable API metadata:** Each parameter type in `fst_params.py`, `base_params.py`, etc. carries both a type constraint (for `@_check_types` validation) and a human-readable description (in the `Annotated` string). This is more than just documentation — it creates a structured, introspectable description of every parameter in the API. The recently merged `describe_api()` method (PR #904) uses method-level introspection; the parameter-level metadata in these `Annotated` aliases could serve as the foundation for deeper introspection — for example, an NLP interface that needs to know not just which method to call, but what parameters each method accepts and what values are valid.

---

*Author: Aswani Sahoo — March 2026*
