# Plan — PER-100 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Fix rustc recursion overflow when deriving Serialize on internally tagged recursive enums by changing newtype serialization in serde_derive; single-repo bug fix with clear behavior stated in ticket.

## Steps

1. [x] **analyst (inline_story)** — Transform the ticket's requirements and acceptance criteria into a structured Given/When/Then story with tech notes for the serde_derive change.
   - focus: Focus on the internally tagged newtype serialization path in serde_derive that causes recursion. Emphasize that only Serialize derivation for internally tagged enums needs fixing, and Deserialize stays unchanged. Note the regression test requirement and the specific fork target.
2. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope is clearly stated: fix serde_derive for internally tagged newtype serialization only, with explicit boundaries on what not to change.
- **designer** — The ticket already specifies the concrete fix location (serde_derive internally tagged newtype path) and acceptance criteria; no file-level design needed before coding.
- **test_architect** — Test requirements are explicitly stated in acceptance criteria: one test_suite test covering three shapes with round-trip verification.
- **critic** — Single inline story produces only one artifact (requirements.md with embedded story), so no cross-document consistency check is needed.
