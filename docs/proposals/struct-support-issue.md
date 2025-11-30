# Issue: Add support for defining structs and accessing their values by attributes

## Overview

Add LLVM IR codegen support for struct types in IRx, enabling users to define structs with named fields and access/modify those fields using attribute syntax. This is inspired by LLVM's [CodeGenTypes.cpp](https://github.com/llvm/llvm-project/blob/main/clang/lib/CodeGen/CodeGenTypes.cpp#L792) approach to struct type layout and field access.

## Background

Currently, IRx supports basic data types (int, float, string, etc.) but lacks support for user-defined composite types like structs. The `astx` library already provides:
- `astx.StructDefStmt` - AST node for struct definitions
- `astx.StructDeclStmt` - AST node for struct declarations

However, IRx does not yet translate these to LLVM IR or support field access operations.

## Implementation Phases

---

### Phase 1: Struct Type Definition

**Goal:** Translate `astx.StructDefStmt` to LLVM IR identified struct types.

**Tasks:**
- [ ] Add `struct_types` dictionary in `LLVMLiteIRVisitor` to track struct name → LLVM `IdentifiedStructType` mapping
- [ ] Add `struct_defs` dictionary to store struct definitions for field lookup
- [ ] Implement `visit(node: astx.StructDefStmt)` to:
  - Create LLVM `IdentifiedStructType` using `module.context.get_identified_type(name)`
  - Map each field's astx type to LLVM type
  - Set struct body with `struct_type.set_body(*field_types)`

**Example LLVM IR output:**
```llvm
%"Point" = type {i32, i32}
```

---

### Phase 2: Struct Type Reference

**Goal:** Allow variables to be declared with struct types.

**Tasks:**
- [ ] Create `StructType` class in `irx.system` to represent struct type references
- [ ] Update `visit(node: astx.VariableDeclaration)` to handle struct types:
  - Allocate struct on stack using `ir_builder.alloca(struct_type)`
  - Track struct type metadata for variable

**Example usage:**
```python
point_type = StructType(struct_name="Point")
point_var = astx.VariableDeclaration(name="p", type_=point_type)
```

---

### Phase 3: Struct Instantiation

**Goal:** Create struct instances with field initializers.

**Tasks:**
- [ ] Create `StructExpr` class for struct instantiation expressions
- [ ] Implement `visit(node: StructExpr)` to:
  - Allocate struct on stack
  - Initialize each field using GEP + store
  - Return pointer to struct

**Example usage:**
```python
point_expr = StructExpr(
    struct_type=point_type,
    field_values={"x": astx.LiteralInt32(10), "y": astx.LiteralInt32(20)}
)
```

---

### Phase 4: Attribute Access (Read)

**Goal:** Access struct field values by name (e.g., `point.x`).

**Tasks:**
- [ ] Create `AttributeExpr` class for field access expressions
- [ ] Implement helper `_get_struct_field_index(struct_name, field_name)` to look up field index
- [ ] Implement `visit(node: AttributeExpr)` to:
  - Get struct pointer from value expression
  - Use GEP to get field pointer: `getelementptr %Struct, ptr %struct_ptr, i32 0, i32 <field_index>`
  - Load field value

**Example LLVM IR:**
```llvm
%x_ptr = getelementptr %"Point", %"Point"* %p, i32 0, i32 0
%x = load i32, i32* %x_ptr
```

---

### Phase 5: Attribute Assignment (Write)

**Goal:** Assign values to struct fields (e.g., `point.x = 42`).

**Tasks:**
- [ ] Create `AttributeAssignment` class for field assignment statements
- [ ] Implement `visit(node: AttributeAssignment)` to:
  - Get struct pointer
  - Use GEP to get field pointer
  - Store new value

---

### Phase 6: Testing & Documentation

**Tasks:**
- [ ] Add unit tests for each phase:
  - `test_struct_definition` - verify struct type in IR
  - `test_struct_instantiation` - verify field initialization
  - `test_struct_attribute_access` - verify GEP + load
  - `test_struct_attribute_assignment` - verify store
  - `test_struct_build_and_run` - end-to-end execution test
- [ ] Update README with struct usage examples
- [ ] Add docstrings to all new classes and methods

---

## Technical Notes

### LLVM Concepts Used
- `IdentifiedStructType` - Named struct types (vs anonymous `LiteralStructType`)
- `GEP (GetElementPointer)` - Access struct fields by index
- `alloca` - Stack allocation for struct instances
- `store/load` - Field value read/write

### Key Files to Modify
- `src/irx/system.py` - Add `StructType`, `StructExpr`, `AttributeExpr`, `AttributeAssignment`
- `src/irx/builders/llvmliteir.py` - Add visitor methods and struct tracking
- `tests/test_struct.py` - New test file

### Dependencies
- Requires `astx >= 0.23.1` (already supports `StructDefStmt`)
- Uses `llvmlite.ir.IdentifiedStructType`

---

## Related

- LLVM CodeGenTypes reference: https://github.com/llvm/llvm-project/blob/main/clang/lib/CodeGen/CodeGenTypes.cpp#L792
- astx StructDefStmt: Already available in astx
