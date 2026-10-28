# CHERI MIPS Test Porting Log

Running log of triage decisions for the MIPS-CHERI tests: what was
ported (and where), what was removed (and why), and anything learned
along the way. Newest decisions first. This file is the durable record;
keep it untracked in the llvm-project checkout.

**Base branch:** `<base-branch>` (the branch that collects the triage
work; every port/remove branch is created from it).

## Summary

| Disposition | Count |
|---|---:|
| Ported | 0 |
| Not ported (erase) | 0 |
| **Total decided** | **0** |

Count files, not commits: a batch entry that covers several files adds
one per file.

## Porting policy: minimal-diff for test content

When porting a test:

**Change** only what's required:
- RUN lines (→ `@…ARGS@` templating in `CHERI-Generic/Inputs/`, or the
  RISC-V invocation used by a sibling test).
- `!DO NOT AUTOGEN!` for crash-only or hand-CHECKed tests.
- Values that depend on capability width (`iCAPRANGE`, `@CAP_BYTES@`, …)
  where a hard-coded MIPS value would be wrong on RISC-V.
- Inert module boilerplate (`; ModuleID`, `source_filename`,
  `target triple`), which the RUN line supplies per target.

**Keep verbatim**, even when a semantically equivalent rewrite exists:
- IR bodies, instruction sequences, type syntax (typed pointers stay
  typed), no-op casts, comments, attribute lists, CHECK prefixes, and
  RUN flags such as `-verify-machineinstrs` or `-relocation-model=static`.

Always regenerate and run the tests rather than assuming.

## Coverage-verification discipline

Before deciding a test is redundant ("covered by X"), verify that X
covers the **same scenario at the same level**. A matching file name or
operation name is not enough. Check:

- **Orderings, immediates, operand types and widths.** A cap `cmpxchg`
  test that only uses `acq_rel` does not cover `seq_cst`; an integer
  `xchg` does not cover a capability `xchg`; sub-legal and legal widths
  take different SelectionDAG paths.
- **Pipeline level.** A crash/ISel regression needs a cover that runs
  the same path (e.g. `-verify-machineinstrs` or the same pass), not an
  asm FileCheck of a similar-looking function.
- **Every case the test pins, including negative ones** (operand
  rejected, diagnostic emitted, relocation not emitted).
- **Evidence.** Name the covering test and line, and run it. Where the
  test guards a fix, revert the fix and check that the cover trips.

## ERASE subcategories

Cite one in every erase entry:

- **ERASE-redundant** — the generic scenario is already covered on
  RISC-V at the same level (verified as above).
- **ERASE-MIPS-ISA: register/encoding constraint** — pins a MIPS
  encoding rule with no RISC-V analogue.
- **ERASE-MIPS-ISA: MIPS-only optimization pass** — pins a pass that
  exists only in the MIPS backend.
- **ERASE-MIPS-ISA: MIPS-only TableGen pattern bug** — pins an ISel
  pattern bug in MIPS-only patterns or operand classes.

Add new subcategories here when a decision doesn't fit, and use them
consistently afterwards.

## Side findings (collect here, file as issues later)

Factual observations made along the way (tooling quirks, latent
issues, missing RISC-V coverage). Don't claim a bug unless the
evidence shows what was intended; otherwise write "intent unclear".

## Decisions (chronological)

<!-- New entries go at the top of this section. Format:

### YYYY-MM-DD — `<repo-relative path of the test>`

**Decision:** Ported. / Not ported (erase, ...). Subcategory: **PORT** / **ERASE-...**.

**Target:** (for PORT) where it went.

**Branch:** `port-...` / `remove-...`.

**Rationale (git archaeology / coverage / empirical):** ...

**Verification.** (for PORT) regenerate + lit results.

**Templating notes.** (for PORT) ...

**Inventory correction:** (only if an inventory exists) ...
-->
