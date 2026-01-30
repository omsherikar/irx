# Issue: Phase 1 - Struct Type Definition

## Title
Add LLVM IR codegen for `StructDefStmt` (struct type definitions)

## Description

Implement the translation of `astx.StructDefStmt` to LLVM IR identified struct types. This is the foundation for struct support in IRx.

### Background
- `astx` already provides `StructDefStmt` for defining structs
- IRx needs to translate this to LLVM's `IdentifiedStructType`
- Inspired by LLVM's [CodeGenTypes.cpp](https://github.com/llvm/llvm-project/blob/main/clang/lib/CodeGen/CodeGenTypes.cpp#L792)

### Tasks
- [ ] Add `struct_types: dict[str, ir.IdentifiedStructType]` in `LLVMLiteIRVisitor` to track struct name → LLVM type mapping
- [ ] Add `struct_defs: dict[str, astx.StructDefStmt]` to store struct definitions for field lookup
- [ ] Implement `visit(node: astx.StructDefStmt)` visitor method:
  - Create LLVM `IdentifiedStructType` using `module.context.get_identified_type(name)`
  - Map each field's astx type to LLVM type using `get_data_type()`
  - Set struct body with `struct_type.set_body(*field_types)`

### Example Input (Python)
```python
struct_def = astx.StructDefStmt(
    name="Point",
    attributes=[
        astx.VariableDeclaration(name="x", type_=astx.Int32()),
        astx.VariableDeclaration(name="y", type_=astx.Int32()),
    ],
)
```

### Expected LLVM IR Output
```llvm
%"Point" = type {i32, i32}
```

### Files to Modify
- `src/irx/builders/llvmliteir.py`

### Acceptance Criteria
- [ ] `StructDefStmt` translates to LLVM `IdentifiedStructType`
- [ ] Struct types are registered and can be looked up by name
- [ ] Unit test `test_struct_definition` passes
