# Issue: Phase 5 - Attribute Assignment (Write)

## Title
Add `AttributeAssignment` for writing struct field values

## Description

Implement struct field assignment (write) using dot notation, e.g., `point.x = 42`. This enables modifying values in struct fields.

### Background
- After Phase 4, we can read struct fields
- Need to write/modify field values in structs
- LLVM uses GEP (GetElementPointer) + store for field assignment

### Tasks
- [ ] Create `AttributeAssignment` class in `src/irx/system.py`:
  - Inherit from `astx.base.StatementType`
  - Store `target: AttributeExpr` - the field to assign to
  - Store `value: astx.Expr` - the value to assign
- [ ] Implement `visit(node: AttributeAssignment)` in `llvmliteir.py`:
  - Visit target to get struct pointer and field info
  - Visit value to get the value to store
  - Determine struct type from pointer
  - Get field index using helper from Phase 4
  - Use GEP to get field pointer
  - Store value to field pointer

### Example Usage
```python
# Assuming 'p' is a Point struct variable
p_var = astx.Variable(name="p")

# Create attribute expression for x field
x_access = AttributeExpr(value=p_var, attr_name="x")

# Assign new value to x field
x_assignment = AttributeAssignment(
    target=x_access,
    value=astx.LiteralInt32(42)
)

main_block.append(x_assignment)
```

### Expected LLVM IR Output
```llvm
%Point.x_ptr = getelementptr %"Point", %"Point"* %p, i32 0, i32 0
store i32 42, i32* %Point.x_ptr
```

### Files to Modify
- `src/irx/system.py` - Add `AttributeAssignment` class
- `src/irx/builders/llvmliteir.py` - Add `AttributeAssignment` visitor

### Dependencies
- Requires Phase 1-4 to be completed

### Acceptance Criteria
- [ ] `AttributeAssignment` can assign values to struct fields
- [ ] Generated LLVM IR correctly stores values
- [ ] Modified values persist and can be read back
- [ ] Unit test `test_struct_attribute_assignment` passes
