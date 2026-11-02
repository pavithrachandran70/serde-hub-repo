# requirements.md

## Overview

Fix the `serde_derive` procedural macro to prevent unaligned reference creation when deriving `Deserialize` for `#[repr(C, packed)]` structs with the `deserialize_in_place` feature enabled. The current implementation fails with compiler error E0793 because it creates `&mut` references to packed struct fields, which may be unaligned.

## Given/When/Then Acceptance Criteria

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

## Implementation Notes

- The fix location is `serde_derive/src/de/struct_.rs` where `InPlaceSeed(&mut self.place.#member)` is generated
- For packed struct fields, replace direct `&mut` reference creation with `ptr::addr_of_mut!` to obtain a raw pointer without creating an unaligned reference
- Detection of packed structs must respect `#[repr(C, packed)]` and `#[repr(packed)]` annotations
- The fix applies only when `deserialize_in_place` is enabled; the existing code path for feature-off remains unchanged

---

## Out of Scope

- Changing deserialization behavior for non-packed types
- Modifying alignment or packing semantics of user structs
- Reformatting unrelated code or editing crates other than `serde_derive` and `test_suite`
- Opening pull request to upstream `serde-rs/serde` (target is `pavithrachandran70/serde`)
