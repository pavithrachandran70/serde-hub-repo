---
key: PER-98
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Fix packed struct Deserialize derive failing with E0793 when deserialize_in_place
  is enabled
---

**Description**

```text
Bug in serde_derive. Reported upstream as https://github.com/serde-rs/serde/issues/3112. Fix it on this fork (https://github.com/pavithrachandran70/serde) and open the pull request here.
## Reproduction
```rust

#[derive(serde_derive::Deserialize)]

#[repr(C, packed)]

pub struct Packed {

    pub tag: u8,
    pub value: u64,
}
```

`Cargo.toml` must depend on this checkout with the feature enabled:

```toml

serde = { path = "/home/pavc/Documents/Serde-product/serde" }

serde_derive = { path = "/home/pavc/Documents/Serde-product/serde_derive", features = ["deserialize_in_place"] }
```

Seen on rustc 1.99.1 with serde / serde_derive 1.0.229. `cargo check` fails with:

```text

error[E0793]: reference to field of packed struct is unaligned
```

The same struct compiles when `deserialize_in_place` is off. Expected result is that `cargo check` succeeds with the feature on.

## Cause

With `deserialize_in_place`, the derive writes `&mut self.place.#member` (`serde_derive/src/de/struct_.rs`, `InPlaceSeed(&mut self.place.#member)`). On a `#[repr(C, packed)]` struct that creates an unaligned reference to `value: u64`.

## Acceptance criteria

* `cargo check` of the packed struct above succeeds with `deserialize_in_place` enabled.
* The same struct still compiles with the feature disabled.
* Add a regression test under `test_suite`. That package already enables `serde_derive/deserialize_in_place`.
* `cd test_suite && cargo +nightly test --features unstable` passes.
* Keep the public `Deserialize` API unchanged.

## Out of scope

* Changing unpacking, alignment, or `Deserialize` for types that are not packed.
* Reformatting the repository or editing crates other than `serde_derive` and `test_suite`.
* Opening the pull request against `serde-rs/serde`. The pull request goes to `pavithrachandran70/serde`.

```
```
