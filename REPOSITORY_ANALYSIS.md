# IRx Repository Analysis — Potential Issues

> **Purpose**: This document identifies bugs, enhancements, and feature gaps in
> the IRx codebase. Each item is written as a ready-to-file GitHub issue. Items
> already tracked in the upstream [arxlang/irx](https://github.com/arxlang/irx)
> repository are **excluded** to avoid duplication.

---

## Bugs

### 1. `RegisterTable` methods crash with `IndexError` on empty stack

**Labels**: `bug`

**Summary**

All `RegisterTable` methods in `src/irx/symbol_table.py` access
`self.stack[-1]` without checking whether the stack is empty. Calling any of
`increase()`, `last`, `pop()`, `redefine()`, or `reset()` on a freshly created
(or fully popped) `RegisterTable` raises an unguarded `IndexError`.

**Affected code** (`src/irx/symbol_table.py`, class `RegisterTable`):

```python
class RegisterTable:
    def __init__(self) -> None:
        self.stack: list[int] = []

    def increase(self, count: int = 1) -> int:
        self.stack[-1] += count   # IndexError if stack is empty
        return self.stack[-1]

    @property
    def last(self) -> int:
        return self.stack[-1]     # IndexError if stack is empty

    def pop(self) -> None:
        self.stack.pop()          # IndexError if stack is empty

    def redefine(self, count: int) -> None:
        self.stack[-1] = count    # IndexError if stack is empty

    def reset(self) -> None:
        self.stack[-1] = 0        # IndexError if stack is empty
```

**Expected behavior**: Clear error message (e.g.,
`RuntimeError("RegisterTable stack is empty")`) instead of raw `IndexError`.

**Suggested fix**: Add a guard at the top of each method:

```python
if not self.stack:
    raise RuntimeError("RegisterTable: cannot operate on an empty stack.")
```

---

### 2. `run_command()` silently masks subprocess errors

**Labels**: `bug`

**Summary**

In `src/irx/builders/base.py` (and duplicated in `llvmliteir.py`),
`run_command()` catches `CalledProcessError` and returns `str(e.returncode)`.
This discards stderr, the failed command, and the exception context, making it
impossible for callers to distinguish between a successful output of `"1"` and a
failure with exit code 1.

**Affected code** (`src/irx/builders/base.py`, lines 31-37):

```python
except subprocess.CalledProcessError as e:
    output = str(e.returncode)  # Masks the actual error
```

**Expected behavior**: Either re-raise the exception or return a structured
result that preserves the error information.

**Suggested fix**:

```python
except subprocess.CalledProcessError as e:
    raise RuntimeError(
        f"Command failed (exit {e.returncode}): {e.stderr or e.output}"
    ) from e
```

---

### 3. Shared mutable class-level defaults in `LLVMLiteIRVisitor`

**Labels**: `bug`

**Summary**

`LLVMLiteIRVisitor` declares `named_values` and `result_stack` as class-level
mutable defaults. While the constructor reinitializes them, this is a known
Python anti-pattern that causes all instances to share the same dict/list if
`__init__` is bypassed or if a subclass forgets to call `super().__init__()`.

**Affected code** (`src/irx/builders/llvmliteir.py`):

```python
class LLVMLiteIRVisitor(BuilderVisitor):
    named_values: dict[str, Any] = {}      # Shared across all instances!
    result_stack: list[ir.Value | ir.Function] = []  # Shared!
```

**Expected behavior**: Only declare instance attributes inside `__init__`.

**Suggested fix**: Remove class-level defaults and keep only the `__init__`
assignments:

```python
class LLVMLiteIRVisitor(BuilderVisitor):
    def __init__(self) -> None:
        self.named_values: dict[str, Any] = {}
        self.result_stack: list[ir.Value | ir.Function] = []
        # ... rest of init
```

---

### 4. Unchecked `result_stack.pop()` in `WhileStmt` and `ForCountLoopStmt` visitors

**Labels**: `bug`

**Summary**

Several locations in `visit(WhileStmt)` and `visit(ForCountLoopStmt)` call
`self.result_stack.pop()` directly instead of using the existing `safe_pop()`
helper. If the stack is empty (e.g., a no-op body or a statement-only branch),
this raises an unhandled `IndexError`.

**Affected code** (`src/irx/builders/llvmliteir.py`):

- `WhileStmt` visitor: condition pop, body pop
- `ForCountLoopStmt` visitor: condition pop, body pop

**Expected behavior**: Use `safe_pop(self.result_stack)` which returns `None`
on empty stack.

**Note**: Related to upstream #145 (enhance `safe_pop()` diagnostics), but this
issue is specifically about **using** `safe_pop()` where raw `pop()` is
currently called.

---

### 5. ASCII encoding used for string literal data instead of UTF-8

**Labels**: `bug`

**Summary**

In the `LiteralString` / `LiteralUTF8String` visitor, string data is encoded
using `bytearray(string_value + "\0", "ascii")`. This crashes with
`UnicodeEncodeError` on any non-ASCII character (e.g., accented letters, emoji,
CJK characters).

**Affected code** (`src/irx/builders/llvmliteir.py`):

```python
string_data.initializer = ir.Constant(
    string_data_type, bytearray(string_value + "\0", "ascii")
)
```

**Expected behavior**: Use `"utf-8"` encoding to support the full Unicode range:

```python
bytearray(string_value + "\0", "utf-8")
```

**Note**: Upstream #208 covers `LiteralUTF8Char` lowering specifically. This
issue is about the separate `LiteralString`/`LiteralUTF8String` code paths.

---

### 6. Unsafe string slicing in `LiteralDateTime` timezone parsing

**Labels**: `bug`

**Summary**

The `LiteralDateTime` visitor parses timezone offsets using
`"-" in time_part[2:]`. While Python's slice on a short string returns an empty
string rather than raising `IndexError`, the logic silently produces incorrect
results for malformed timestamps with very short time parts (e.g., `"T1"` or
`"T"`). The `-` check on `time_part[2:]` would yield `False` even when the
input is invalid, allowing bad data to pass through without validation.

**Affected code** (`src/irx/builders/llvmliteir.py`):

```python
if time_part.endswith("Z") or "+" in time_part or "-" in time_part[2:]:
```

**Expected behavior**: Validate `time_part` length before slicing, or use a
more robust parsing approach.

---

## Enhancements

### 7. Replace generic `Exception` with `NotImplementedError` in abstract methods

**Labels**: `enhancement`

**Summary**

`BuilderVisitor.translate()` in `src/irx/builders/base.py` raises a bare
`Exception("Not implemented yet.")` instead of `NotImplementedError`. This
makes it harder for IDEs and linters to identify unimplemented abstract methods.

**Affected code** (`src/irx/builders/base.py`):

```python
def translate(self, expr: astx.AST) -> str:
    raise Exception("Not implemented yet.")
```

**Suggested fix**:

```python
def translate(self, expr: astx.AST) -> str:
    raise NotImplementedError("Subclasses must implement translate().")
```

---

### 8. Hardcoded module name `"Arx"` should be configurable

**Labels**: `enhancement`

**Summary**

In `src/irx/builders/llvmliteir.py`, the LLVM module is always created with the
name `"Arx"`:

```python
self._llvm.module = ir.module.Module("Arx")
```

Users should be able to specify a custom module name when creating a builder,
especially for projects that embed IRx in larger compilation pipelines.

**Suggested fix**: Accept an optional `module_name` parameter in the builder
constructor, defaulting to `"Arx"`.

---

### 9. Extract duplicated condition-to-boolean conversion logic

**Labels**: `enhancement`, `refactor`

**Summary**

Both `visit(IfStmt)` and `visit(WhileStmt)` contain nearly identical logic to
convert a condition value to a boolean (comparing against zero for integers,
against 0.0 for floats). This should be extracted into a shared helper method
like `_ensure_boolean(cond_val) -> ir.Value`.

**Benefit**: Reduces code duplication, ensures consistent behavior, and makes it
easier to add new condition types in the future.

---

### 10. Add `__repr__` and `__str__` to `RegisterTable` for debugging

**Labels**: `enhancement`

**Summary**

`RegisterTable` in `src/irx/symbol_table.py` has no string representation. When
debugging codegen issues, developers cannot easily inspect the stack state.

**Suggested fix**:

```python
def __repr__(self) -> str:
    return f"RegisterTable(stack={self.stack})"
```

---

### 11. Standardize exception types across visitor methods

**Labels**: `enhancement`, `code-quality`

**Summary**

Visitor methods use a mix of `Exception`, `ValueError`, `TypeError`, and
`RuntimeError` inconsistently. For example:

- `LiteralDateTime` raises `ValueError` for invalid formats
- Other literal visitors raise generic `Exception` for similar conditions
- Some raise `TypeError` for unsupported types

**Suggested fix**: Define a small hierarchy of IRx-specific exceptions (e.g.,
`IRxCodegenError`, `IRxTypeError`, `IRxValidationError`) and use them
consistently.

---

### 12. Module-level side effects in `tools/typing.py`

**Labels**: `enhancement`, `code-quality`

**Summary**

`src/irx/tools/typing.py` mutates global typeguard configuration at import
time:

```python
global_config.forward_ref_policy = ForwardRefPolicy.IGNORE
global_config.collection_check_strategy = CollectionCheckStrategy.ALL_ITEMS
```

This affects any other library that uses typeguard in the same process.

**Suggested fix**: Provide a `configure_typechecking()` function that users can
call explicitly, or scope the configuration to IRx's own decorators.

---

## Test Improvements

### 13. Add error-path tests for `RegisterTable`

**Labels**: `testing`

**Summary**

`tests/test_symbol.py` thoroughly tests happy-path behavior but has no tests
for error conditions. Empty-stack operations should be tested to ensure clear
error messages instead of raw `IndexError` crashes.

**Missing test cases**:

- `increase()` on empty stack
- `last` on empty stack
- `pop()` on empty stack
- `redefine()` on empty stack
- `reset()` on empty stack
- Negative values passed to `increase()`

---

### 14. Missing comparison operator coverage in conditional tests

**Labels**: `testing`

**Summary**

`tests/test_if_stmt.py` and `tests/test_binary_op.py` only exercise `>` and
`<` comparison operators. The following operators are never tested in
conditional contexts:

- `==` (equality)
- `!=` (inequality)
- `>=` (greater-or-equal)
- `<=` (less-or-equal)

If any of these operators have lowering bugs, the current test suite would not
catch them.

---

### 15. No overflow/boundary tests for type casting

**Labels**: `testing`

**Summary**

`tests/test_cast.py` uses the value `42` for all cast tests. This misses
important edge cases:

- **Overflow**: `Int32(300)` → `Int8` (should wrap/truncate)
- **Negative values**: `Int32(-1)` → unsigned types
- **Boundary values**: `Int32(2147483647)` → `Int16`
- **Precision loss**: `Float64(3.141592653589793)` → `Float32` → `Float64`
- **Special float values**: `NaN`, `Inf`, `-0.0`

---

### 16. Translation (IR output) tests are disabled across all test files

**Labels**: `testing`

**Summary**

Every test file has the `"translate"` action commented out in its parametrize
decorators. This means the generated LLVM IR is never validated for correctness
or structure — only that the final binary builds and runs.

**Example** (appears in nearly every test file):

```python
@pytest.mark.parametrize(
    "action,suffix",
    [
        # ("translate", "test_xyz.ll"),  # <-- always commented out
        ("build", ""),
    ],
)
```

**Impact**: IR regression bugs (incorrect instructions, missing terminators,
invalid types) are only caught when they happen to cause a build or runtime
failure, not when the IR itself is malformed.

**Suggested fix**: Re-enable translation tests and add `.ll` reference files, or
at minimum validate that the IR parses with `llvm.parse_assembly()`.

---

## CI / Infrastructure

### 17. CI uses outdated `actions/checkout@v3`

**Labels**: `ci`, `maintenance`

**Summary**

`.github/workflows/main.yaml` uses `actions/checkout@v3` which is deprecated.
GitHub recommends `actions/checkout@v4` for Node 20 compatibility and security
updates.

---

### 18. No coverage reporting or upload in CI

**Labels**: `ci`, `enhancement`

**Summary**

The CI pipeline computes code coverage (via `pytest-cov` with
`--cov-fail-under=80`) but never uploads the results. Coverage trends, PR
annotations, and historical tracking are unavailable.

**Suggested fix**: Add codecov or coveralls integration, or at minimum upload
the coverage report as a workflow artifact.

---

### 19. Missing `[tool.coverage]` configuration in `pyproject.toml`

**Labels**: `enhancement`, `testing`

**Summary**

`pytest-cov` and `coverage` are installed as dev dependencies, but
`pyproject.toml` has no `[tool.coverage.run]` or `[tool.coverage.report]`
section. This means coverage uses default settings without branch coverage,
source filtering, or display options.

**Suggested addition**:

```toml
[tool.coverage.run]
branch = true
source = ["src/irx"]
omit = ["*/tests/*"]

[tool.coverage.report]
precision = 2
show_missing = true
```

---

### 20. macOS CI testing is disabled without tracking issue

**Labels**: `ci`, `platform`

**Summary**

In `.github/workflows/main.yaml`, macOS is commented out of the test matrix
with the note: *"we need to enable the code generation for macos on ci"*. There
is no tracking issue for this, which means it may be forgotten.

**Note**: Upstream has #58 for macOS CI support, but this fork should track
whether fork-specific changes affect macOS.

---

## Summary Table

| # | Title | Type | Priority | Files |
|---|-------|------|----------|-------|
| 1 | RegisterTable crashes on empty stack | Bug | High | `symbol_table.py` |
| 2 | run_command() masks subprocess errors | Bug | High | `builders/base.py` |
| 3 | Shared mutable class defaults in visitor | Bug | Medium | `builders/llvmliteir.py` |
| 4 | Unchecked pop() in loop visitors | Bug | Medium | `builders/llvmliteir.py` |
| 5 | ASCII encoding for string literals | Bug | Medium | `builders/llvmliteir.py` |
| 6 | Unsafe slicing in DateTime parsing | Bug | Low | `builders/llvmliteir.py` |
| 7 | Use NotImplementedError in abstract methods | Enhancement | Low | `builders/base.py` |
| 8 | Hardcoded module name "Arx" | Enhancement | Low | `builders/llvmliteir.py` |
| 9 | Extract duplicated condition logic | Enhancement | Medium | `builders/llvmliteir.py` |
| 10 | Add __repr__ to RegisterTable | Enhancement | Low | `symbol_table.py` |
| 11 | Standardize exception types | Enhancement | Medium | `builders/llvmliteir.py` |
| 12 | Module-level side effects in typing | Enhancement | Low | `tools/typing.py` |
| 13 | Error-path tests for RegisterTable | Testing | High | `tests/test_symbol.py` |
| 14 | Missing comparison operator tests | Testing | Medium | `tests/test_if_stmt.py` |
| 15 | No overflow tests for casting | Testing | Medium | `tests/test_cast.py` |
| 16 | Translation tests disabled | Testing | High | All test files |
| 17 | Outdated actions/checkout@v3 | CI | Low | `.github/workflows/` |
| 18 | No coverage upload in CI | CI | Medium | `.github/workflows/` |
| 19 | Missing coverage config | Testing | Low | `pyproject.toml` |
| 20 | macOS CI disabled without tracking | CI | Low | `.github/workflows/` |
