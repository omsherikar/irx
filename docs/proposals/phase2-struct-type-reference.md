# Issue: Phase 2 - Struct Type Reference

## Title
Add `StructType` class for struct variable declarations

## Description

Allow variables to be declared with struct types by creating a `StructType` class that references defined structs.

### Background
- After Phase 1, struct types exist in LLVM IR
- Need a way to declare variables with struct types
- Requires a type class that can be used with `astx.VariableDeclaration`

### Tasks
- [ ] Create `StructType` class in `src/irx/system.py`:
  - Inherit from `astx.DataType`
  - Store `struct_name: str` to reference the struct definition
  - Optional `struct_def` reference for direct access
- [ ] Update `visit(node: astx.VariableDeclaration)` in `llvmliteir.py`:
  - Check if `type_` is `StructType`
  - Allocate struct on stack using `ir_builder.alloca(struct_type)`
  - Track struct type metadata for the variable

### Example Usage
```python
# Reference the Point struct defined in Phase 1
point_type = StructType(struct_name="Point")

# Declare a variable of struct type
point_var = astx.VariableDeclaration(name="p", type_=point_type)
```

### Expected LLVM IR Output
```llvm
%p = alloca %"Point"
```

### Files to Modify
- `src/irx/system.py` - Add `StructType` class
- `src/irx/builders/llvmliteir.py` - Update `VariableDeclaration` visitor

### Dependencies
- Requires Phase 1 (struct type definition) to be completed

### Acceptance Criteria
- [ ] `StructType` class can reference defined structs
- [ ] Variables can be declared with struct types
- [ ] Stack allocation generates correct LLVM IR
- [ ] Unit test `test_struct_variable_declaration` passes
