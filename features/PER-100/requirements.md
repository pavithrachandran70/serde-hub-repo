# Requirements — PER-100

**Title:** Fix internally tagged recursive enums: derived Serialize overflows or hangs rustc

**Type:** Bug fix

**Repository:** app (https://github.com/pavithrachandran70/serde)

**Base branch:** master

**Test command:** `cd test_suite && cargo +nightly test --features unstable`

**Lint command:** `cd serde && cargo +nightly clippy --features rc,unstable -- -Dclippy::all -Dclippy::pedantic`

---

## Problem Statement

Deriving `Serialize` on an internally tagged enum that contains itself causes rustc to fail with recursion overflow (E0275) or hang when the recursion limit is raised. The same enum with only `Deserialize` derived compiles successfully. This blocks users from serializing recursive data structures with internal tagging.

**Example failing code:**

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Expr {
    Eq { field: String },
    And(Vec<Expr>),
    Not(Box<Expr>),
}
```

---

## Acceptance Criteria

### AC1: Recursive enum compiles

**Given** an internally tagged enum with recursive newtype variants (e.g., `And(Vec<Expr>)`, `Not(Box<Expr>)`)  
**When** `Serialize` is derived on that enum  
**Then** `cargo check` succeeds with no E0275 recursion overflow error

**Verification:**
- Compile the `Expr` enum shown above in a test file
- Assert that `cargo check` exits 0
- Assert no E0275 error appears in compiler output

---

### AC2: Serialization round-trips all shapes

**Given** an internally tagged recursive enum with `Serialize` and `Deserialize` derived  
**When** serializing and deserializing instances of struct variants, tuple variants with `Vec`, and tuple variants with `Box`  
**Then** all shapes round-trip correctly, including one level of nesting

**Verification:**
- Serialize `Expr::Eq { field: "x".into() }` and deserialize; assert equality
- Serialize `Expr::And(vec![Expr::Eq { field: "y".into() }])` and deserialize; assert equality
- Serialize `Expr::Not(Box::new(Expr::Eq { field: "z".into() }))` and deserialize; assert equality
- Serialize `Expr::And(vec![Expr::Not(Box::new(Expr::Eq { field: "a".into() }))])` (nested) and deserialize; assert equality

---

### AC3: Struct variants still work

**Given** an internally tagged enum with struct-like variants  
**When** serializing and deserializing struct variants  
**Then** existing behavior is unchanged

**Verification:**
- Serialize and deserialize `Expr::Eq { field: "test" }` and assert round-trip
- Compare serialized JSON structure to expected format with `"type": "Eq"` present

---

### AC4: Non-recursive internally tagged newtype still works

**Given** an internally tagged enum with a non-recursive newtype variant (e.g., `Wrapper(Inner)` where `Inner` does not contain the enum)  
**When** serializing that variant  
**Then** serialization succeeds with existing behavior preserved

**Verification:**
- Define an enum with a non-recursive newtype variant
- Serialize an instance of that variant
- Assert serialized output matches expected format
- Deserialize and assert equality

---

### AC5: Deserialize behavior unchanged

**Given** an internally tagged recursive enum with `Deserialize` derived  
**When** deserializing JSON matching the enum's structure  
**Then** deserialization behavior is identical to before the fix

**Verification:**
- Deserialize a JSON string into `Expr::And(vec![...])`
- Assert deserialized structure matches expected value
- Compare deserialization behavior before and after fix (use snapshot or existing test baseline)

---

### AC6: Regression test added

**Given** a new test in `test_suite`  
**When** the test suite runs  
**Then** a test covering the three shapes (`Eq`, `And`, `Not`) with internal tagging passes

**Verification:**
- Locate new test file or test function in `test_suite/tests/`
- Run `cd test_suite && cargo +nightly test --features unstable`
- Assert test passes
- Assert test covers struct variant, `Vec` newtype, and `Box` newtype

---

### AC7: Test suite passes

**Given** the complete serde codebase with the fix applied  
**When** running the full test suite  
**Then** `cd test_suite && cargo +nightly test --features unstable` exits 0 with all tests passing

**Verification:**
- Execute `cd test_suite && cargo +nightly test --features unstable`
- Assert exit code 0
- Assert no test failures in output

---

## Out of Scope

- Changing externally tagged enums
- Changing adjacently tagged enums
- Changing untagged enums
- Modifying public `Serialize` or `Deserialize` traits
- Emitting a nicer compiler error without fixing the root cause
- Instructing users to raise `recursion_limit`
- Opening PR on `serde-rs/serde` (target is `pavithrachandran70/serde`)

---

## Technical Notes

**Focus area:** Internally tagged newtype serialization path in `serde_derive`

**What changes:** Only the code path that derives `Serialize` for internally tagged enums with newtype variants

**What stays the same:** `Deserialize` derivation for this enum pattern, and all other tagging modes

**Test location:** `test_suite/tests/`

**PR target:** `pavithrachandran70/serde`, branch `master`

---

## Success Signal

The `Expr` enum example compiles cleanly, all three variant shapes serialize and deserialize correctly with internal tagging, and the test suite passes with a new regression test covering this fix.
