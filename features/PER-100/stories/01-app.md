# PER-100 — Fix internally tagged recursive enums: derived Serialize overflows or hangs rustc

**Type:** bug  
**Repo:** app  
**Status:** draft  
**Created:** 2025-01-24

---

## Problem

Deriving `Serialize` on an internally tagged enum that contains itself causes rustc to fail with recursion overflow (E0275) or hang when the recursion limit is raised. The same enum with only `Deserialize` derived compiles successfully.

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

**Expected:** `cargo check` succeeds. `serde_json` round-trips `Eq`, `And(Vec<Expr>)`, and `Not(Box<Expr>)`, including `And` containing `Not` containing `Eq`.

---

## Scope

- Change internally tagged newtype serialization in `serde_derive` only
- Add regression test under `test_suite/tests/`
- Do not change externally tagged, adjacently tagged, or untagged enums
- Do not change public `Serialize` or `Deserialize` traits
- Do not only emit a nicer error or tell users to raise `recursion_limit`
- Open PR on `pavithrachandran70/serde`, not `serde-rs/serde`

---

## Technical Spec

**Root cause:** The internally tagged newtype serialization path in `serde_derive` generates code that causes infinite recursion during trait resolution when the newtype variant contains the enum itself (directly via `Box` or indirectly via `Vec`).

**Fix location:** `serde_derive/src/ser.rs` (or equivalent), specifically the code path handling `#[serde(tag = "...")]` with newtype variants.

**Current behavior:** When deriving `Serialize` for an internally tagged newtype variant like `And(Vec<Expr>)`, the macro generates code that attempts to serialize the inner type in a way that triggers recursive trait resolution on `Expr: Serialize`, leading to overflow.

**Required change:** Modify the generated code for internally tagged newtype variants to serialize the wrapped value without triggering recursive trait bounds during compilation. This likely involves:

1. Detecting newtype variants in internally tagged enums during macro expansion
2. Generating serialization code that delegates to the inner type's `Serialize` impl without creating a recursive trait bound cycle
3. Ensuring the `"type"` tag is still correctly emitted in the serialized output

**Constraints:**

- Only `Serialize` derivation for internally tagged enums needs fixing
- `Deserialize` behavior must remain unchanged (it already works)
- Struct-variant forms (like `Eq { field: String }`) must continue to work
- Non-recursive internally tagged newtypes (like `Wrapper(Inner)` where `Inner` does not contain the enum) must serialize as they do today

**Files likely affected:**

- `serde_derive/src/ser.rs` — serialization code generation for internally tagged enums
- `test_suite/tests/test_gen.rs` or a new test file — regression test

**Test requirements:**

- Add a test in `test_suite/tests/` covering the three shapes: struct variant (`Eq`), `Vec` newtype (`And`), `Box` newtype (`Not`)
- Test must verify round-trip serialization via `serde_json::to_string` and `serde_json::from_str`
- Test must include one level of nesting (e.g., `And` containing `Not` containing `Eq`)

---

## Acceptance Criteria

### AC1: Recursive enum compiles

**Given** the `Expr` enum shown above with `#[derive(Serialize, Deserialize)]` and `#[serde(tag = "type")]`  
**When** running `cargo check`  
**Then** compilation succeeds with no E0275 recursion overflow error

**Verify:**

```rust
// In test file or example
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Expr {
    Eq { field: String },
    And(Vec<Expr>),
    Not(Box<Expr>),
}
// cargo check must exit 0, no E0275
```

---

### AC2: Serialization round-trips all shapes

**Given** an instance of each variant shape  
**When** serializing via `serde_json::to_string` and deserializing via `serde_json::from_str`  
**Then** all shapes round-trip correctly, including one level of nesting

**Verify:**

```rust
let eq = Expr::Eq { field: "x".into() };
let json = serde_json::to_string(&eq).unwrap();
let roundtrip: Expr = serde_json::from_str(&json).unwrap();
assert_eq!(eq, roundtrip);

let and = Expr::And(vec![Expr::Eq { field: "y".into() }]);
let json = serde_json::to_string(&and).unwrap();
let roundtrip: Expr = serde_json::from_str(&json).unwrap();
assert_eq!(and, roundtrip);

let not = Expr::Not(Box::new(Expr::Eq { field: "z".into() }));
let json = serde_json::to_string(&not).unwrap();
let roundtrip: Expr = serde_json::from_str(&json).unwrap();
assert_eq!(not, roundtrip);

let nested = Expr::And(vec![Expr::Not(Box::new(Expr::Eq { field: "a".into() }))]);
let json = serde_json::to_string(&nested).unwrap();
let roundtrip: Expr = serde_json::from_str(&json).unwrap();
assert_eq!(nested, roundtrip);
```

---

### AC3: Struct variants still work

**Given** the struct-like variant `Eq { field: String }`  
**When** serializing and deserializing  
**Then** existing behavior is unchanged, with `"type": "Eq"` present in JSON

**Verify:**

```rust
let eq = Expr::Eq { field: "test".into() };
let json = serde_json::to_string(&eq).unwrap();
assert!(json.contains(r#""type":"Eq"#));
let roundtrip: Expr = serde_json::from_str(&json).unwrap();
assert_eq!(eq, roundtrip);
```

---

### AC4: Non-recursive newtype still works

**Given** an internally tagged enum with a non-recursive newtype variant  
**When** serializing that variant  
**Then** serialization succeeds with existing format preserved

**Verify:**

```rust
#[derive(Serialize, Deserialize, PartialEq, Debug)]
#[serde(tag = "kind")]
enum Container {
    Simple(String),
}

let simple = Container::Simple("value".into());
let json = serde_json::to_string(&simple).unwrap();
let roundtrip: Container = serde_json::from_str(&json).unwrap();
assert_eq!(simple, roundtrip);
```

---

### AC5: Deserialize behavior unchanged

**Given** JSON matching the `Expr` structure  
**When** deserializing via `serde_json::from_str`  
**Then** deserialization behavior is identical to before the fix

**Verify:**

```rust
let json = r#"{"type":"And","And":[{"type":"Eq","field":"test"}]}"#;
let expr: Expr = serde_json::from_str(json).unwrap();
// Assert deserialized structure matches expected
match expr {
    Expr::And(vec) => {
        assert_eq!(vec.len(), 1);
        match &vec[0] {
            Expr::Eq { field } => assert_eq!(field, "test"),
            _ => panic!("wrong variant"),
        }
    }
    _ => panic!("wrong top-level variant"),
}
```

---

### AC6: Regression test added

**Given** a new test in `test_suite/tests/`  
**When** running `cd test_suite && cargo +nightly test --features unstable`  
**Then** a test covering `Eq`, `And`, and `Not` with internal tagging passes

**Verify:**

- New test file exists at `test_suite/tests/test_internally_tagged_recursive.rs` (or similar)
- Test function covers struct variant, `Vec<Self>` newtype, and `Box<Self>` newtype
- `cd test_suite && cargo +nightly test --features unstable` includes this test in output and reports it passing

---

### AC7: Full test suite passes

**Given** the complete fix applied  
**When** running `cd test_suite && cargo +nightly test --features unstable`  
**Then** all tests pass with exit code 0

**Verify:**

```bash
cd test_suite && cargo +nightly test --features unstable
# Exit code 0, no test failures in output
```

---

## Notes

**Planner focus:** The internally tagged newtype serialization path in `serde_derive` causes recursion. Only `Serialize` derivation for internally tagged enums needs fixing; `Deserialize` stays unchanged. Regression test required. PR target is `pavithrachandran70/serde`, branch `master`.

**Open questions:** None — scope, behavior, and test requirements are explicitly stated in ticket.
