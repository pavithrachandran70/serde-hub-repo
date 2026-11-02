# PER-98: Fix packed struct Deserialize derive failing with E0793

**Type:** bug  
**Epic:** (standalone)  
**Status:** planned  
**Assignee:** (unassigned)  
**Estimate:** 2 points  

---

## Description

Fix `serde_derive` procedural macro to prevent unaligned reference creation when deriving `Deserialize` for `#[repr(C, packed)]` structs with the `deserialize_in_place` feature enabled. The current implementation in `serde_derive/src/de/struct_.rs` generates `&mut self.place.#member` for in-place deserialization, which creates unaligned references on packed struct fields and fails compilation with error E0793.

**Root cause:** The `InPlaceSeed` code path generates direct mutable references to packed struct fields. On `#[repr(C, packed)]` types, fields like `value: u64` may not be aligned, making `&mut self.place.value` undefined behavior per Rust's reference validity rules.

**Solution approach:** Replace direct `&mut` reference creation with `ptr::addr_of_mut!` for packed struct fields, obtaining a raw pointer without triggering the unaligned reference error. Detection logic must recognize `#[repr(C, packed)]` and `#[repr(packed)]` attributes on the struct being derived.

---

## Reproduction Case

```rust
#[derive(serde_derive::Deserialize)]
#[repr(C, packed)]
pub struct Packed {
    pub tag: u8,
    pub value: u64,
}
```

With `Cargo.toml`:
```toml
serde = { path = "/home/pavc/Documents/Serde-product/serde" }
serde_derive = { path = "/home/pavc/Documents/Serde-product/serde_derive", features = ["deserialize_in_place"] }
```

Running `cargo check` fails with:
```
error[E0793]: reference to field of packed struct is unaligned
```

Expected: `cargo check` succeeds with `deserialize_in_place` enabled.

---

## Acceptance Criteria

### AC1: Packed struct compiles with deserialize_in_place enabled

**Given** a struct annotated with `#[derive(serde_derive::Deserialize)]` and `#[repr(C, packed)]`  
**And** the struct contains fields of different alignment requirements (e.g., `u8` and `u64`)  
**And** `serde_derive` is built with the `deserialize_in_place` feature enabled  
**When** the user runs `cargo check` on code using this struct  
**Then** compilation succeeds with no E0793 errors

**Verification:** Run `cargo check` on the reproduction case from the ticket with `deserialize_in_place` enabled; build must succeed.

---

### AC2: Packed struct still compiles with deserialize_in_place disabled

**Given** the same packed struct from AC1  
**And** `serde_derive` is built with the `deserialize_in_place` feature disabled  
**When** the user runs `cargo check`  
**Then** compilation succeeds as it does today

**Verification:** Run `cargo check` on the reproduction case with the feature disabled; build must succeed.

---

### AC3: Regression test added and passing

**Given** a new test case in `test_suite` that derives `Deserialize` for a packed struct  
**And** `test_suite` already enables `serde_derive/deserialize_in_place`  
**When** the user runs `cd test_suite && cargo +nightly test --features unstable`  
**Then** all tests pass, including the new regression test

**Verification:** The new test must exercise the packed struct deserialization path and pass in CI.

---

### AC4: Public Deserialize API unchanged

**Given** existing code that derives `Deserialize` (packed or not)  
**When** the fix is applied  
**Then** the generated `Deserialize` implementation has the same public signature and behavior  
**And** no breaking changes are introduced to the `Deserialize` trait contract

**Verification:** Existing `test_suite` tests must all pass without modification.

---

## Technical Specification

### Modified Files

**`serde_derive/src/de/struct_.rs`**

Locate the `InPlaceSeed` code generation block that produces:
```rust
InPlaceSeed(&mut self.place.#member)
```

Modify the code path to:

1. **Detect packed structs:** Check the struct's `repr` attributes for `packed` or `packed(N)`. This information is available in the derive macro's input `syn::DeriveInput` via `attrs` and can be parsed with `syn::Attribute::parse_meta()`.

2. **Conditional reference creation:**
   - **For packed structs:** Generate code using `ptr::addr_of_mut!` to obtain a raw mutable pointer:
     ```rust
     InPlaceSeed(unsafe { &mut *::core::ptr::addr_of_mut!((*self.place).#member) })
     ```
   - **For non-packed structs:** Keep existing code:
     ```rust
     InPlaceSeed(&mut self.place.#member)
     ```

3. **Safety justification:** The `unsafe` block is sound because:
   - `self.place` is a valid mutable reference to the struct
   - `addr_of_mut!` creates a raw pointer without creating an intermediate reference, avoiding UB
   - The dereferencing and re-borrowing happens only after obtaining the correctly-aligned raw pointer
   - The caller's contract ensures `self.place` lives long enough for the deserialization

### New Test File

**`test_suite/tests/test_derive_deserialize_packed.rs`**

Add a regression test:

```rust
#![cfg(feature = "deserialize_in_place")]

use serde_derive::Deserialize;

#[test]
fn test_packed_struct_deserialize_in_place() {
    #[derive(Deserialize, Debug, PartialEq)]
    #[repr(C, packed)]
    struct Packed {
        tag: u8,
        value: u64,
    }

    let json = r#"{"tag": 42, "value": 1234567890}"#;
    let deserialized: Packed = serde_json::from_str(json).unwrap();
    
    assert_eq!(deserialized.tag, 42);
    assert_eq!(deserialized.value, 1234567890);
}

#[test]
fn test_packed_tuple_struct() {
    #[derive(Deserialize, Debug, PartialEq)]
    #[repr(packed)]
    struct PackedTuple(u8, u64, u16);

    let json = r#"[1, 999, 42]"#;
    let deserialized: PackedTuple = serde_json::from_str(json).unwrap();
    
    assert_eq!(deserialized.0, 1);
    assert_eq!(deserialized.1, 999);
    assert_eq!(deserialized.2, 42);
}
```

### Implementation Constraints

- **Feature gate:** All changes are conditional on `#[cfg(feature = "deserialize_in_place")]`
- **Rust version:** Must work on Rust 1.51+ (when `ptr::addr_of_mut!` stabilized)
- **No public API changes:** Generated code changes are internal to the derive macro; user-facing `Deserialize` trait remains identical
- **Scope:** Only modify `serde_derive/src/de/struct_.rs` and add tests in `test_suite/tests/`

---

## Verification Steps

1. **Build reproduction case:**
   ```bash
   cargo check
   ```
   Expected: Success (currently fails with E0793)

2. **Run full test suite with feature enabled:**
   ```bash
   cd test_suite
   cargo +nightly test --features unstable,deserialize_in_place
   ```
   Expected: All tests pass, including new `test_derive_deserialize_packed.rs`

3. **Run full test suite with feature disabled:**
   ```bash
   cd test_suite
   cargo +nightly test --features unstable
   ```
   Expected: All tests pass (packed struct test skipped due to `#![cfg(feature = "deserialize_in_place")]`)

4. **Lint check:**
   ```bash
   cd serde
   cargo +nightly clippy --features rc,unstable -- -Dclippy::all -Dclippy::pedantic
   ```
   Expected: No new warnings

---

## Out of Scope

- Changing deserialization behavior for non-packed types
- Modifying alignment or packing semantics of user structs  
- Reformatting unrelated code or editing crates other than `serde_derive` and `test_suite`
- Opening pull request to upstream `serde-rs/serde` (target is `pavithrachandran70/serde`)

---

## Notes

- **Upstream issue:** https://github.com/serde-rs/serde/issues/3112
- **Target repo:** https://github.com/pavithrachandran70/serde
- **Base branch:** `master`
- The fix must preserve existing behavior for all non-packed structs; regression risk is mitigated by feature-gating and comprehensive test coverage in `test_suite`
