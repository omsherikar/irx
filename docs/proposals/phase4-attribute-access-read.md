# Issue: Phase 4 - Attribute Access (Read)

## Title
Add `AttributeExpr` for reading struct field values

## Description

Implement struct field access (read) using dot notation, e.g., `point.x`. This enables reading values from struct fields.

### Background
- After Phase 3, we can create struct instances
- Need to read field values from structs
- LLVM uses GEP (GetElementPointer) + load for field access

### Tasks
- [ ] Create `AttributeExpr` class in `src/irx/system.py`:
  - Store `value: astx.Expr` - the struct expression
  - Store `attr_name: str` - the field name to access
- [ ] Add helper `_get_struct_field_index(struct_name, field_name)` in `llvmliteir.py`:
  - Look up struct definition
  - Return field index for the given field name
- [ ] Implement `visit(node: AttributeExpr)` in `llvmliteir.py`:
  - Visit value expression to get struct pointer
  - Determine struct type from pointer
  - Get field index using helper
  - Use GEP: `getelementptr %Struct, ptr %struct_ptr, i32 0, i32 <field_index>`
  - Load field value

### Example Usage
```python
# Assuming 'p' is a Point struct variable
p_var = astx.Variable(name="p")

# Access x field
x_access = AttributeExpr(value=p_var, attr_name="x")

# Use in return statement
main_block.append(astx.FunctionReturn(x_access))
```

### Expected LLVM IR Output
```llvm
%Point.x_ptr = getelementptr %"Point", %"Point"* %p, i32 0, i32 0
%Point.x = load i32, i32* %Point.x_ptr
```

### Files to Modify
- `src/irx/system.py` - Add `AttributeExpr` class
- `src/irx/builders/llvmliteir.py` - Add `AttributeExpr` visitor and helper

### Dependencies
- Requires Phase 1-3 to be completed

### Acceptance Criteria
- [ ] `AttributeExpr` can access struct fields by name
- [ ] Field index is correctly determined from struct definition
- [ ] Generated LLVM IR uses correct GEP indices
- [ ] Loaded values can be used in expressions
- [ ] Unit test `test_struct_attribute_access` passes
