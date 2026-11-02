# Plan — PER-98 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Fix serde_derive macro to avoid unaligned references in packed structs when deserialize_in_place is enabled; single-repo bug fix with clear reproduction and acceptance criteria.

## Steps

1. [ ] **analyst (inline_story)** — Convert the detailed bug report into a structured story with implementation guidance.
   - focus: Focus on the unsafe reference creation in serde_derive/src/de/struct_.rs InPlaceSeed code; the fix must use ptr::addr_of_mut! for packed struct fields to avoid creating unaligned references.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no other spec documents to cross-check for consistency.
