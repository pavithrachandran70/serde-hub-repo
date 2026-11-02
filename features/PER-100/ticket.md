---
key: PER-100
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: 'Fix internally tagged recursive enums: derived Serialize overflows or hangs
  rustc'
---

Deriving Serialize on an internally tagged enum that holds itself fails to compile. The same enum with only Deserialize compiles. Raising the recursion limit makes rustc hang instead of finishing.

#\[derive(Serialize, Deserialize)\]

#\[serde(tag = "type")\]

enum Expr {

```
Eq { field: String },
And(Vec<Expr>),
Not(Box<Expr>),
```

}

// cargo check fails with E0275

Expected behavior. cargo check succeeds at the default recursion limit. serde_json round-trips Eq, And(Vec<Expr>), and Not(Box<Expr>), including And containing Not containing Eq. The same shapes written as struct variants still compile and round-trip. A non-recursive newtype such as Wrapper(Inner) still serializes as it does today.

Scope. Change internally tagged newtype serialization in serde_derive only. Add a regression test under test_suite. Do not change externally tagged, adjacently tagged, or untagged enums. Do not change the public Serialize or Deserialize traits. Do not only emit a nicer error, and do not tell the user to raise recursion_limit. Open the pull request on pavithrachandran70/serde, not serde-rs/serde.

Acceptance criteria.

* cargo check of the Expr enum above succeeds. No E0275.
* to_string / from_str round-trips the three shapes, including one level of nesting.
* Struct-variant forms and a non-recursive internally tagged newtype still work.
* Deserialize behavior for this enum is unchanged.
* A new test_suite test covers the three shapes.
* cd test_suite && cargo +nightly test --features unstable passes.
