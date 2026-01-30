# Issue: Phase 6 - Testing & Documentation

## Title
Add comprehensive tests and documentation for struct support

## Description

Create complete test coverage and documentation for the struct support feature implemented in Phases 1-5.

### Background
- Phases 1-5 implement the core struct functionality
- Need comprehensive tests to ensure correctness
- Need documentation to help users utilize the feature

### Tasks

#### Testing
- [ ] Create `tests/test_struct.py` with the following tests:
  - `test_struct_definition` - verify struct type definition in IR
  - `test_struct_variable_declaration` - verify struct variable allocation
  - `test_struct_instantiation` - verify field initialization
  - `test_struct_attribute_access` - verify GEP + load
  - `test_struct_attribute_assignment` - verify GEP + store
  - `test_struct_build_and_run` - end-to-end execution test
  - `test_nested_structs` - structs containing other structs (optional)
  - `test_struct_in_function_args` - passing structs to functions (optional)

#### Documentation
- [ ] Update `README.md` with struct usage examples
- [ ] Add docstrings to all new classes:
  - `StructType`
  - `StructExpr`
  - `AttributeExpr`
  - `AttributeAssignment`
- [ ] Add docstrings to new visitor methods
- [ ] Document LLVM IR patterns in code comments

### Example Test
```python
def test_struct_build_and_run():
    """Test building and running a program with struct operations."""
    builder = LLVMLiteIR()
    module = builder.module()

    # Define Point struct
    struct_def = astx.StructDefStmt(
        name="Point",
        attributes=[
            astx.VariableDeclaration(name="x", type_=astx.Int32()),
            astx.VariableDeclaration(name="y", type_=astx.Int32()),
        ],
    )
    module.block.append(struct_def)

    # Create main function that returns sum of fields
    main_proto = astx.FunctionPrototype(
        name="main", args=astx.Arguments(), return_type=astx.Int32()
    )
    main_block = astx.Block()

    # Create and initialize Point
    point_type = StructType(struct_name="Point")
    point_expr = StructExpr(
        struct_type=point_type,
        field_values={"x": astx.LiteralInt32(10), "y": astx.LiteralInt32(20)}
    )
    point_var = astx.VariableDeclaration(name="p", type_=point_type, value=point_expr)
    main_block.append(point_var)

    # Return x + y (should be 30)
    p_var = astx.Variable(name="p")
    x_access = AttributeExpr(value=p_var, attr_name="x")
    y_access = AttributeExpr(value=p_var, attr_name="y")
    main_block.append(astx.FunctionReturn(x_access + y_access))

    main_fn = astx.FunctionDef(prototype=main_proto, body=main_block)
    module.block.append(main_fn)

    # Build and run - expect exit code 30
    check_result("build", builder, module, expected_output="30")
```

### Files to Modify/Create
- `tests/test_struct.py` - New test file
- `README.md` - Add struct usage section
- `src/irx/system.py` - Add docstrings
- `src/irx/builders/llvmliteir.py` - Add docstrings

### Dependencies
- Requires Phases 1-5 to be completed

### Acceptance Criteria
- [ ] All unit tests pass
- [ ] Test coverage for struct functionality ≥ 90%
- [ ] Documentation is clear and complete
- [ ] Example code in README works correctly
- [ ] CI/CD pipeline passes
