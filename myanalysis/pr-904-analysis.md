# PR #904 Analysis: Adding `describe_api()` Method for API Introspection

**PR:** [#904](https://github.com/malariagen/malariagen-data-python/pull/904)
**Issue:** [#903](https://github.com/malariagen/malariagen-data-python/issues/903)
**Author:** Mandeep Singh ([@mandeepsingh2007](https://github.com/mandeepsingh2007))
**Reviewer/Merger:** Jon Brenas ([@jonbrenas](https://github.com/jonbrenas))
**Merged:** 2026-02-25
**Diff:** +244 lines, −0 lines across 3 files (entirely additive, no existing code modified)

---

## 1. What the PR Does

This PR adds a new public method `describe_api()` to the `malariagen-data-python` package. When called, it returns a pandas DataFrame listing every public method available on the API object (e.g., `ag3` or `af1`), along with a one-line summary extracted from the docstring and a category label (`"data"`, `"analysis"`, or `"plot"`). It also supports an optional `category` parameter to filter the output to only methods of a given type.

The stated motivation is foundational work for the GSoC 2026 project on natural-language interfaces: an NLP system needs programmatic API discovery before it can map user queries to the right methods.

### Jon's review comment

Jon approved the PR with the comment:

> *"LGTM. I am not sure it is as useful for users as it is for automated systems but it doesn't hurt to have the option."*

This is a significant observation. Jon sees `describe_api()` as primarily useful for *automated* systems rather than human users, which aligns exactly with its intended role as a foundation for NLP-based query translation.

In a follow-up comment after approval, Jon added an important architectural clarification:

> *"The outcome of Project 3 would have to be a tool built on the API for data access but separate from it... Some parts of the proposal may require changes in the API but not all of them. I think the content of this PR is a valuable step towards a solution to Project 3, hence the approval."*

This is the clearest articulation of where `describe_api()` sits in the broader picture: it lives *inside* the API as a discovery endpoint, while the NLP interface itself must live *outside* the API, calling it as a user would. The PR does not start building the NLP tool; it creates the programmatic hook that such a tool would use. The boundary between the two is important: the API provides data and methods, the external tool provides the natural-language layer.

---

## 2. Files Changed

| File | Lines | Purpose |
|------|-------|---------|
| [`malariagen_data/anoph/describe.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/malariagen_data/anoph/describe.py) | +115 (new file) | The `AnophelesDescribe` mixin class containing `describe_api()` and two helper methods |
| [`malariagen_data/anopheles.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/malariagen_data/anopheles.py#L47) | +2 | Import `AnophelesDescribe` and insert it into the `AnophelesDataResource` inheritance list |
| [`tests/anoph/test_describe.py`](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/tests/anoph/test_describe.py) | +127 (new file) | 14 unit tests covering the method's behaviour, edge cases, and helper functions |

---

## 3. Architectural Decision: The Mixin Pattern

### How it fits into the existing class hierarchy

The `malariagen-data-python` codebase uses cooperative multiple inheritance to compose the main `AnophelesDataResource` class from many focused mixin classes. The comment in [`anopheles.py` (lines 63–77)](https://github.com/malariagen/malariagen-data-python/blob/c269768ff0ba89d0755298757ea215ee297fb430/malariagen_data/anopheles.py#L63-L77) explains this explicitly:

> *"We are in the process of breaking up the AnophelesDataResource class into multiple parent classes... This is work in progress..."*

It also references [issue #366](https://github.com/malariagen/malariagen-data-python/issues/366) and links to resources on C3 linearization and Python's MRO.

PR #904 follows this established pattern exactly: it creates a new `AnophelesDescribe` mixin that inherits from `AnophelesBase` and is inserted into the `AnophelesDataResource` inheritance list. This means `describe_api()` becomes available on every API object (`ag3`, `af1`, etc.) without modifying any existing class.

### MRO placement matters

In the `AnophelesDataResource` class definition, `AnophelesDescribe` is placed **immediately above `AnophelesBase`**:

```python
class AnophelesDataResource(
    AnophelesDipClustAnalysis,
    AnophelesHapClustAnalysis,
    # ... 14 other mixins ...
    AnophelesDescribe,       # ← inserted here
    AnophelesBase,
    AnophelesPhenotypeData,
):
```

This positioning is important because of Python's C3 linearization algorithm, which determines the method resolution order (MRO). Since `AnophelesDescribe` inherits from `AnophelesBase`, it must appear *before* `AnophelesBase` in the parent list; otherwise Python would encounter `AnophelesBase` twice in conflicting positions and raise a `TypeError`. The `super().__init__(**kwargs)` call in `AnophelesBase.__init__()` is designed to work cooperatively, passing remaining keyword arguments up the chain so that all mixins get properly initialised.

### Why a separate mixin (instead of adding to an existing class)

The method could have been added directly to `AnophelesBase`, which would have been simpler (no new file, no MRO change). But the codebase convention is clear: each mixin handles a distinct functional area. `AnophelesBase` handles configuration, caching, filesystem initialisation, and GCS location checking. API introspection is a different concern: it is about *discovering* the API, not about *running* data queries. Keeping it in its own mixin follows the separation-of-concerns pattern established by `AnophelesFstAnalysis`, `AnophelesPca`, `AnophelesHapData`, etc.

---

## 4. The Three Core Components of `describe_api()`

### 4a. Method discovery: walking `dir(self)`

```python
for name in sorted(dir(self)):
    if name.startswith("_"):
        continue
    attr = getattr(type(self), name, None)
    if attr is None:
        continue
    if isinstance(attr, property):
        continue
    if not callable(attr):
        continue
```

**What this does:** Iterates over all attributes of the instance, sorted alphabetically. It filters out:
- Private/dunder methods (names starting with `_`)
- `None` attributes (names that exist in `dir()` but not as class attributes)
- Properties (e.g., `contigs`, `site_mask_ids`, which are data attributes, not callable methods)
- Non-callable attributes

**A subtle choice: `getattr(type(self), name, None)` vs `getattr(self, name)`.** The code looks up the attribute on the *class* (`type(self)`), not on the *instance*. This is intentional: looking up on the class avoids triggering property getters or descriptors that might have side effects or require data access. For method introspection, you want the unbound function object, not a bound method or computed value.

### 4b. Summary extraction: `_extract_summary()`

```python
@staticmethod
def _extract_summary(method) -> str:
    docstring = inspect.getdoc(method)
    if not docstring:
        return ""
    for line in docstring.strip().splitlines():
        line = line.strip()
        if line:
            return line
    return ""
```

**What this does:** Uses `inspect.getdoc()` to retrieve the docstring of a method, then returns the first non-empty line as a short summary.

**Why `inspect.getdoc()` and not `method.__doc__`?** `inspect.getdoc()` does two important things that `__doc__` does not:
1. It cleans up indentation: docstrings in methods are typically indented to match the method body, and `inspect.getdoc()` strips this leading whitespace consistently
2. It resolves inherited docstrings: if a method overrides a parent method but doesn't define its own docstring, `inspect.getdoc()` will walk up the MRO to find one

**Interaction with `numpydoc_decorator`:** Most methods in the codebase use the `@doc(summary="...", ...)` decorator from `numpydoc_decorator`. This decorator generates a formatted docstring from the keyword arguments and assigns it to the function's `__doc__` attribute. The `summary` parameter becomes the first line of the generated docstring. Because `_extract_summary()` takes the first non-empty line, it reliably recovers the original summary text that the author wrote in the `@doc()` call. This works because of a convention (the `@doc` decorator always puts the summary first) rather than because of any explicit contract between `describe.py` and `numpydoc_decorator`.

### 4c. Method categorisation: `_categorize_method()`

```python
@staticmethod
def _categorize_method(name: str) -> str:
    if name.startswith("plot_"):
        return "plot"
    data_prefixes = (
        "sample_", "snp_", "hap_", "cnv_", "genome_",
        "open_", "lookup_", "read_", "general_",
        "sequence_", "cohorts_", "aim_", "gene_",
    )
    if name.startswith(data_prefixes):
        return "data"
    return "analysis"
```

**What this does:** Assigns a category to each method based solely on its name prefix. Plot methods start with `plot_`. Data access methods start with one of 13 known prefixes. Everything else defaults to `"analysis"`.

**Why this approach was chosen and its trade-offs:**

*Strengths:*
- Simple and fast: no need to inspect method signatures, decorators, or return types
- Deterministic: the same method name always gets the same category
- Covers the current codebase well: every existing public method follows the naming convention

*Weaknesses:*
- **Hardcoded prefixes are fragile.** If a new data access method is added with a prefix not in the tuple (e.g., `phenotype_data()`), it would be misclassified as `"analysis"`. The tuple would need manual updating. There is no mechanism to warn a developer that they should add their new prefix to this list.
- **The "analysis" default is a catch-all.** Methods like `pairwise_average_fst`, `average_fst`, `fst_gwss`, `diversity_stats`, and `cohort_diversity_stats` are all classified as "analysis" not because they positively match a pattern, but because they don't match `"plot_"` or any data prefix. This means any non-standard method (including `describe_api` itself) falls into "analysis" by default.
- **`describe_api()` is categorised as "analysis"** even though it is neither data access, analysis, nor plotting: it is introspection/meta. A more granular category scheme (e.g., adding `"meta"` or `"utility"`) could be more accurate, but would add complexity for a single method.

*Alternative approaches that could have been used:*
1. **Decorator-based tagging**: e.g., `@data_method`, `@plot_method`, `@analysis_method` decorators that attach metadata. More robust, but would require modifying every existing method in the codebase, a much larger change.
2. **Docstring-based detection**: parse the `@doc()` parameters for category hints. Would couple categorisation to documentation format.
3. **Return type inspection**: plot methods return `Figure` objects, data methods return DataFrames/arrays. Would require calling `get_type_hints()` on each method, which could be expensive and error-prone with complex annotations.

The prefix-based approach was the right pragmatic choice for a first implementation: it works correctly for all current methods and can be extended later if needed.

---

## 5. The Category Filter Logic

```python
if category is not None:
    valid_categories = {"data", "analysis", "plot"}
    if category not in valid_categories:
        raise ValueError(
            f"Invalid category: {category!r}. "
            f"Must be one of {valid_categories}."
        )
    df = df[df["category"] == category].reset_index(drop=True)
```

**Design note:** The validation happens *after* building the full DataFrame, not before. This means all methods are always discovered and categorised, and then the unwanted rows are filtered out. An alternative would be to skip non-matching methods during the `for name in sorted(dir(self))` loop, which would avoid building rows that are immediately discarded. The current approach is simpler to read and test, and performance is not a concern — the method list is small (dozens of methods, not thousands).

**The `category` parameter uses `Optional[str]`, not `Literal`:** Unlike the `annotation` parameter in `fst_params.py` (which uses `Annotated[Optional[Literal[...]]]`), `category` is typed as a plain `Optional[str]` with runtime validation via an explicit `if/raise`. This means the `@_check_types` decorator cannot catch invalid category values at the type level — the function itself handles it. The PR does not use `@_check_types` at all; it only uses `@doc()` for documentation. This is a notable departure: every other public method in the codebase (verified across `fst.py`, `anopheles.py`, and all mixin files) stacks **both** `@_check_types` and `@doc()`. `describe_api()` is the only public method that omits `@_check_types` entirely.

Using `Literal["data", "analysis", "plot"]` with `@_check_types` would have been more consistent with the rest of the codebase and would provide the same validation with less code. However, for a method whose primary audience is automated systems (as Jon noted), the explicit `ValueError` with a helpful error message may be preferable.

---

## 6. Testing Strategy: 14 Tests

The test file (`test_describe.py`) creates separate `AnophelesDescribe` fixtures for both the Ag3 and Af1 simulators, using `pytest_cases`'s `@parametrize_with_cases` decorator. This means each test runs twice, once against an Ag3-shaped API and once against an Af1-shaped API, producing 14 test results from 7 test functions.

| Test | What It Verifies |
|------|------------------|
| `test_describe_api_returns_dataframe` | Output is a DataFrame with columns `method`, `summary`, `category`; has at least one row |
| `test_describe_api_no_private_methods` | No method name in the output starts with `_` |
| `test_describe_api_category_filter` | Filtering by each of `"data"`, `"analysis"`, `"plot"` returns only methods of that category |
| `test_describe_api_invalid_category` | Passing `category="invalid"` raises `ValueError` with the message `"Invalid category"` |
| `test_describe_api_known_methods` | `describe_api` itself and `sample_sets` (from `AnophelesBase`) appear in the output |
| `test_describe_api_summaries_not_empty` | At least some methods have non-empty summary strings |
| `test_categorize_method` (standalone) | Direct unit test of `_categorize_method` with 8 method name examples |
| `test_extract_summary` (standalone) | Direct unit test of `_extract_summary` with a dummy function (with and without docstring) |

**What the tests do well:**
- Cross-species coverage (Ag3 + Af1) ensures the method works regardless of which data resource is being used
- The `test_known_methods` test verifies that `describe_api` lists itself and a known `AnophelesBase` method — this catches MRO integration issues
- The standalone `test_categorize_method` and `test_extract_summary` tests exercise the helper functions in isolation, independent of any API fixture

**What the tests do not cover:**
- **Completeness**: there is no assertion that the output contains *all* public methods. A method could be silently excluded (e.g., if `getattr(type(self), name, None)` returns `None` for some reason) and no test would catch it.
- **Category correctness**: the test only checks that filtering works, not that specific methods have the correct category. For example, there is no assertion that `pairwise_average_fst` is categorised as `"analysis"` when called through the full `AnophelesDataResource`, rather than the standalone `AnophelesDescribe` fixture.
- **Summary accuracy**: no test checks that a specific method's summary matches its `@doc(summary="...")` value.

These gaps are acceptable for a first implementation: the method is primarily about discovery, not about guaranteeing perfect metadata.

---

## 7. What `describe_api()` Provides, and What It Does Not

This is the most important section for understanding the PR's role in the broader NLP interface project.

### What it provides (method-level discovery)

| Column | Example Value | Source |
|--------|---------------|--------|
| `method` | `"pairwise_average_fst"` | `dir(self)` filtered by name/callable checks |
| `summary` | `"Compute pairwise average Hudson's Fst between a set of specified cohorts."` | First line of docstring via `inspect.getdoc()` |
| `category` | `"analysis"` | Name prefix matching |

This is enough for a basic question: *"Which method should I call to compare genetic differentiation between populations?"* An NLP system can match the user's intent against the summary descriptions to find `pairwise_average_fst`.

### What it does NOT provide (parameter-level detail)

For an NLP system to go beyond method discovery and actually *generate executable code*, it needs to know:

| Missing Information | Why It Matters | Where It Exists in the Codebase |
|---|---|---|
| **Parameter names** | The NLP system needs to know that `pairwise_average_fst` takes `region`, `cohorts`, `sample_sets`, etc. | `inspect.signature()` or `get_type_hints()` on each method |
| **Parameter types** | It needs to know that `region` is a string, `cohorts` is a string or dict, `min_cohort_size` is an int | The `Annotated[..., ...]` type aliases in `fst_params.py`, `base_params.py`, etc. |
| **Valid values for constrained parameters** | For parameters like `annotation`, the valid set is `{None, "standard error", "Z score", "lower triangle"}` | The `Literal[...]` types inside the `Annotated` wrappers |
| **Parameter descriptions** | Human-readable descriptions of what each parameter does (e.g., `"Minimum samples per cohort"`) | The string component of each `Annotated[type, "description"]` alias |
| **Default values** | Whether a parameter is optional and what its default is | `inspect.signature().parameters[name].default` |
| **Return type** | What the method returns (DataFrame, tuple, Figure, etc.) | Type hints on the method return annotation |

All of this information *already exists* in the codebase — it is encoded in the `Annotated` type aliases, the `@doc()` decorator parameters, and the standard Python function signatures. `describe_api()` surfaces the first layer (method name + summary + category). The next layer — parameter-level introspection — is what a natural-language interface would need to generate valid, executable API calls.

### A concrete example of the gap

Imagine a user asks: *"Show me Fst between populations in Kenya grouped by country."*

With `describe_api()` alone, an NLP system can:
1. ✅ Match the query to `pairwise_average_fst` (via the summary text)

But to generate the actual call, it also needs:
2. ❌ Know that `region` is required (and a contig name like `"3L"`)
3. ❌ Know that `cohorts="country"` is how you group by country
4. ❌ Know that `sample_query="country == 'Kenya'"` is how you filter by country
5. ❌ Know that `min_cohort_size=10` is a sensible default

Steps 2–5 require parameter-level introspection that `describe_api()` does not provide. This is the gap between method-level discovery and code generation, and it is exactly the space where a natural-language interface project would build.

---

## 8. Broader Observations

### The `Annotated` type alias pattern as machine-readable API metadata

Working through both this PR and PR #969, I noticed that the codebase's `Annotated[type, "description"]` pattern (used in `fst_params.py`, `base_params.py`, and other `*_params.py` files) is unusually rich for a Python project. Each parameter's type alias carries:
- A Python type (for `@_check_types` runtime validation via `typeguard`)
- A human-readable description string (for `numpydoc_decorator`'s `@doc` to generate docs)
- For constrained parameters, a `Literal[...]` type that enumerates all valid values

This is effectively a **structured schema** for the entire API, embedded in the type system. `describe_api()` surfaces the outermost layer of this schema (method names and summaries). But the parameter-level layer — types, descriptions, valid values, defaults — is all extractable at runtime using `typing.get_type_hints()` and `inspect.signature()`. An NLP interface could build on this existing infrastructure rather than needing to create a separate API specification.

### Jon's review comment as a design signal

Jon's observation that `describe_api()` is *"useful for automated systems"* rather than human users is not just a passing comment; it reflects a genuine architectural property. Human users of `malariagen-data-python` typically work in Jupyter notebooks where tab-completion, `?` help, and the training course materials provide method discovery. A flat DataFrame of all methods is less useful than these interactive tools. But for an automated system, whether an NLP query translator, a code generator, or a testing framework, a structured, programmatically queryable method registry is exactly the right interface.

---

*Author: Aswani Sahoo, March 2026*
