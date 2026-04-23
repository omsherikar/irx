# Issue: Phase 3 - Struct Instantiation

## Title
Add `StructExpr` for creating struct instances with field initializers

## Description

Create a way to instantiate structs with initial field values using a new `StructExpr` expression class.

### Background
- After Phase 2, we can declare struct variables
- Need a way to initialize struct fields when creating instances
- Similar to constructor calls in other languages

### Tasks
- [ ] Create `StructExpr` class in `src/irx/system.py`:
  - Store `struct_type: StructType` reference
  - Store `field_values: Dict[str, astx.Expr]` for field initializers
  - Generate unique name for the instance
- [ ] Implement `visit(node: StructExpr)` in `llvmliteir.py`:
  - Allocate struct on stack
  - For each field in struct definition:
    - Get value from `field_values` or use default
    - Use GEP to get field pointer
    - Store value to field
  - Return pointer to struct

### Example Usage
```python
point_type = StructType(struct_name="Point")

point_expr = StructExpr(
    struct_type=point_type,
    field_values={
        "x": astx.LiteralInt32(10),
        "y": astx.LiteralInt32(20),
    }
)

# Use in variable declaration
point_var = astx.VariableDeclaration(
    name="p", 
    type_=point_type, 
    value=point_expr
)
```

### Expected LLVM IR Output
```llvm
%struct_inst_0 = alloca %"Point"
%struct_inst_0.x_ptr = getelementptr %"Point", %"Point"* %struct_inst_0, i32 0, i32 0
store i32 10, i32* %struct_inst_0.x_ptr
%struct_inst_0.y_ptr = getelementptr %"Point", %"Point"* %struct_inst_0, i32 0, i32 1
store i32 20, i32* %struct_inst_0.y_ptr
```

### Files to Modify
- `src/irx/system.py` - Add `StructExpr` class
- `src/irx/builders/llvmliteir.py` - Add `StructExpr` visitor

### Dependencies
- Requires Phase 1 (struct type definition)
- Requires Phase 2 (struct type reference)

### Acceptance Criteria
- [ ] `StructExpr` can create struct instances with field values
- [ ] Default values from struct definition are used when not provided
- [ ] Generated LLVM IR correctly initializes all fields
- [ ] Unit test `test_struct_instantiation` passes
