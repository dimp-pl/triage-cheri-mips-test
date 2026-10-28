---
name: triage-cheri-mips-test
description: Triage a MIPS-CHERI test — CodeGen (llvm/test/CodeGen/Mips/cheri/), MC/assembler (llvm/test/MC/Mips/cheri/), Clang frontend (clang/test/.../cheri/), or LLD/linker (lld/test/ELF/cheri/) — and decide whether to port it to its RV-CHERI home (CHERI-Generic / RISCV CHERI / MC RISCV / lld RISCV / clang) or erase it. Investigate what the test does, do git archaeology on the bug-fix history, check RV CHERI coverage, verify empirically, present a verdict with evidence for user approval, then implement (port or erase) on a dedicated branch with porting-log update and commit message. Use when the user asks to "proceed with", "triage", "port", or "erase" a specific MIPS-CHERI test file under any of those trees, or when the user asks to "look at" or "analyse" one.
---

# Triage a CHERI-MIPS test

## Invocation

This skill can be invoked either implicitly (from natural-language
requests like "proceed with `<path>`", "triage `<path>`", or
"look at `<path>`") **or** explicitly with the test path as an
argument:

```
/triage-cheri-mips-test ./llvm/test/CodeGen/Mips/cheri/<basename>
/triage-cheri-mips-test ./llvm/test/MC/Mips/cheri/<basename>
```

The test may live in any of the MIPS-CHERI test trees (see "Test
layers" below): CodeGen (`llvm/test/CodeGen/Mips/cheri/`), MC
(`llvm/test/MC/Mips/cheri/`), Clang (`clang/test/.../cheri/`), or LLD
(`lld/test/ELF/cheri/`, incl. `mips/`).

When the skill is invoked with an argument:

- The first non-flag token is the **test path** to triage. It may be
  given relative to the repo root (e.g.
  `./llvm/test/MC/Mips/cheri/blockaddress-relocations.s`) or as an
  absolute path — use it **verbatim**.
- A bare basename (e.g. `cheri128.ll`) is assumed under
  `llvm/test/CodeGen/Mips/cheri/` **only as a fallback**; if it does
  not exist there, search the other trees
  (`fd <basename> llvm/test/MC/Mips lld/test clang/test`) and confirm
  the match with the user before proceeding.
- Normalise to a repo-relative form before proceeding.
- Verify the file exists; if not, ask the user to confirm or
  correct the path before starting work.

When the skill is invoked without an argument, ask the user which
test file to triage.

## Working files (created on first use)

The skill keeps two working files in the root of the llvm-project
checkout. They are **not** shipped with the skill: create them from the
templates in this skill's `templates/` directory the first time they are
needed, then keep them up to date. Leave them untracked (suggest adding
them to `.git/info/exclude`); they are the triager's notes, not part of
the LLVM tree.

- **`CHERI_MIPS_TEST_PORTING_LOG.md`** (required) — the durable record
  of every decision, plus the policy sections this skill refers to
  ("Porting policy: minimal-diff", "Coverage-verification discipline",
  "ERASE subcategories") and the "Side findings" list. If it doesn't
  exist, create it from `templates/CHERI_MIPS_TEST_PORTING_LOG.md`
  before the first verdict, and ask the user for the **base branch**
  (the branch that collects the triage work) to record in its header.
- **`CHERI_MIPS_TEST_INVENTORY.md`** (optional) — one row per MIPS-CHERI
  test with a **preliminary** verdict from a quick first pass. Create it
  from `templates/CHERI_MIPS_TEST_INVENTORY.md` only if the user asks
  for an inventory (e.g. to plan or track the whole job). When a test is
  decided, update its row with the final decision.

## Goal

The MIPS backend will be removed. For every MIPS-CHERI test — whether
CodeGen, MC/assembler, Clang frontend, or LLD/linker (see "Test
layers" below) — decide whether the **coverage it provides should be
preserved post-MIPS-removal** — by porting it to a location chosen to
fit what the test actually exercises (see "Choosing the port
destination" below) — or whether it can be safely dropped (already
covered on RV, or fundamentally MIPS-only).

`CHERI-Generic/Inputs/` is the most common destination *for CodeGen
tests*, but it is **not** a default to reach for reflexively, and it
is the wrong home for non-CodeGen layers. It is the right home only
for multi-arch *codegen/asm* tests. IR-level transform tests, RV-
specific codegen, MC/assembler tests, LLD tests, and Clang frontend
tests each have a more appropriate home — picking it is part of the
triage, not an afterthought.

If the inventory (`CHERI_MIPS_TEST_INVENTORY.md`) exists, its verdicts
are **preliminary** — always re-analyse from first principles.

## Test layers — the triage shape differs by layer

The detailed flow below (Steps 1–7) is written for **CodeGen** tests,
which are the bulk of the work and the richest case. The other layers
reuse the *same* spine — investigate → git archaeology → check the
porting log → classify → check RV coverage → verify → present → stop —
but differ in two things: **what "the scenario" means** and **where a
PORT lands**. Identify the layer first (from the path), then apply its
deltas.

| Layer | Path | What "the scenario" is | RV home for a PORT |
|---|---|---|---|
| **CodeGen** | `llvm/test/CodeGen/Mips/cheri/` | An IR-level behaviour lowered to asm. MIPS asm in CHECK lines is a **cosmetic lowering artifact** (the "biggest trap"). | `CHERI-Generic/Inputs/` (codegen), `Transforms/InstCombine\|InstSimplify/CHERI/` (IR fold), `RISCV/cheri/` (RV-only) |
| **MC** | `llvm/test/MC/Mips/cheri/` | The **assembler encoding / relocation itself**. The MIPS instruction bytes and `R_MIPS_CHERI_*` reloc names **are the subject under test**, not cosmetic — see the MC inversion below. | `llvm/test/MC/RISCV/cheri/` |
| **Clang** | `clang/test/CodeGen/cheri/`, `clang/test/Analysis/Checkers/CHERI/` | Frontend / static-analyzer behaviour (IR emitted, diagnostics). | the same `clang/test/...` dir, retargeted to an RV triple |
| **LLD** | `lld/test/ELF/cheri/`, `lld/test/ELF/cheri/mips/` | **Linker** behaviour — cap-relocs, cap-table layout, ELF reloc processing. Like MC, reloc *encodings* are per-ISA; *linker concepts* (cap-table, `__cap_relocs`) may be generic with RV counterparts. | `lld/test/ELF/cheri/riscv/` (19 RV tests already live here) |

### The MC inversion (read alongside "Biggest trap" below)

For **CodeGen**, the recurring trap is *over-erasing* because MIPS asm
in CHECK lines looks MIPS-only when the underlying scenario is
generic — the asm is a cosmetic artifact of lowering.

For **MC**, this **inverts**: there is no IR being lowered. The
encoding bytes and the `R_MIPS_CHERI_*` reloc names *are* exactly what
the test asserts, so they are load-bearing and inherently per-ISA. RV
CHERI uses an entirely different relocation family (`R_RISCV_CHERI_*`,
no `CAPTAB20`-style reloc) tested independently under
`MC/RISCV/cheri/`. The same applies to the reloc-encoding half of LLD
tests.

The MC/LLD rule is therefore: **`ERASE-MIPS-encoding-specific` *unless*
the construct is a generic-MC/linker concept AND RV lacks a
counterpart → then PORT to `MC/RISCV/cheri/` (or
`lld/test/ELF/cheri/riscv/`).** It is *not* "MC = always erase":
- `ERASE-redundant` when an RV counterpart already exists (e.g.
  `MC/Mips/cheri/chericap-expr.s` — `MC/RISCV/cheri/chericap-expr.s`
  already covers it).
- `ERASE-MIPS-encoding-specific` when the encoding/reloc is genuinely
  MIPS-only with no generic analog (e.g. `R_MIPS_CHERI_CAPTAB20`,
  `clcbi`/`cscbi` encodings).
- `PORT` (to `MC/RISCV/cheri/`) when the construct is generic-MC and
  RV has no equivalent yet — validated precedent:
  `chericap-size-not-known-yet.s` → `MC/RISCV/cheri/chericap-unknown-size.s`.

Beware: this is exactly the class where a **first-pass "PORT
because the concept is generic" verdict is systematically wrong** — MC
reloc/encoding tests are per-ISA, so re-verify against the RV tree
rather than trusting a first-pass PORT.

**Not every MC test is an encoding/reloc test — classify the sub-type
before applying the inversion.** The inversion above ("the bytes are
the subject → per-ISA → mostly ERASE") holds for **encoding /
relocation** tests. It does **not** hold for **parser / operand-class /
directive-semantics** tests — which operands an instruction accepts or
rejects, `.chericap` expression folding, register-name handling,
assembler diagnostics. Those assert *target-agnostic assembler
behaviour*; the MIPS mnemonic/register spelling is the cosmetic
artifact, exactly like MIPS asm in a CodeGen CHECK line — so the
**CodeGen over-erasing trap applies here, not the inversion**. For this
sub-type, verify whether RV pins the *behaviour* (every accept / reject
/ fold case the test asserts, **including the negative ones**) before
erasing. Validated **PORT-slim** precedent: `ddc-vs-cnull.s` — a MIPS
`$c0`-legacy parser test whose diagnostics + `-cheri-strict-c0-asm`
flag + removed `cfromptr` half were genuinely MIPS-only, but whose
NULL-vs-DDC operand-class distinction (null accepted / `$ddc` rejected
for `cincoffset`/`csetaddr`) was a *verified-unpinned* RV gap → ported
slim, not erased. We almost erased it on the "MC mostly erases" prior.

### Consult the CHERI ISA spec for instruction/encoding-existence questions

When the verdict turns on an **ISA-level fact** — does this
instruction/encoding/reloc still exist, was it removed or renamed,
does it have an RV-CHERI counterpart, is the RV token the same
instruction or just a same-named alias — **do not answer from memory.**
Model recall of per-version ISA changelogs is unreliable and has been
wrong here (a confident "`cjr` was removed" that the spec contradicted).
The authoritative source is the CHERI ISA specification's own version
changelog:

- **Repo:** `CTSRD-CHERI/cheri-specification` (GitHub). The per-version
  changelog lives in `app-versions.tex` (through ~v7) and
  `app-versions-8-0.tex` (v8, the last CHERI-MIPS spec, UCAM-CL-TR-951).
  Fetch the raw file and grep for the mnemonic
  (`raw.githubusercontent.com/CTSRD-CHERI/cheri-specification/main/app-versions-8-0.tex`).
- **PDF:** CHERI ISAv7 = UCAM-CL-TR-907; v8 = UCAM-CL-TR-951
  (`cl.cam.ac.uk/techreports/`).

This matters most for **MC/LLD** tests, where the instruction/encoding
*is* the subject. Two recurring traps the spec resolves:

- **Same mnemonic ≠ same instruction.** A token appearing in
  `MC/RISCV/cheri/` may be an *assembly alias* (e.g. CHERI-RISC-V
  documents `cjr`/`cjalr`/`cret` as aliases of cap-mode `JALR`, not
  distinct opcodes) rather than a real counterpart to the MIPS opcode.
  An alias hit is **not** RV coverage of the MIPS encoding → still
  `ERASE-MIPS-encoding-specific`.
- **Removed/reworked families.** The cap control-transfer family churned
  hard: `CCall`/`CReturn` → renamed `CInvoke` → in v8 replaced by a
  selector-less call. An instruction the MIPS test pins may no longer
  exist in the ISA at all — reinforcing ERASE, never PORT.

Cite the specific changelog line + version in the verdict, the same way
you cite a covering RV test.

## Mental model — the question that matters

Every MIPS-CHERI test will:

- Use MIPS-specific RUN flags (`%cheri_purecap_llc`,
  `-cheri-cap-table-abi=*`, `-disable-cheri-addressing-mode-folder`,
  etc.).
- Have MIPS-specific instructions in CHECK lines (`cgetpccincoffset`,
  `clcbi`, `cfromddc`, `$c11`, `$c17`, `$ddc`, …).
- Have MIPS-specific calling-convention details
  (`$c3`/`$c4` arg regs, N64 ABI stack layout).

These are **expected cosmetic artifacts of the test running on the
MIPS backend** — they tell you what MIPS happens to emit. They are
*not* evidence that the test is MIPS-only. The real question is:

> *Is the underlying scenario the test exercises also relevant on RV
> CHERI, and is it already covered there?*

Three outcomes follow:

1. **Scenario generic AND not yet covered on RV → PORT.**
   The port runs on MIPS / RV32 / RV64 in `CHERI-Generic/`; per-arch
   CHECK lines or templated values (`@CAP_BYTES@`, `iCAPRANGE`,
   `@IF-MIPS@`, …) handle output differences. The MIPS asm in the
   original CHECK lines is replaced by per-arch asm at regeneration
   time — it was never load-bearing for the bug class.

2. **Scenario generic AND already covered on RV → ERASE-redundant.**
   The MIPS test duplicates coverage that already survives MIPS
   removal. Be specific about *which* RV test covers it, at *which*
   pipeline level (IR-fold, ISel, asm).

3. **Scenario is genuinely MIPS-only → ERASE-MIPS-only-feature.**
   The feature or bug class has no RV analog. Concretely:
   - MIPS-specific intrinsic / builtin (`int_mips_*`,
     `__builtin_mips_*`).
   - MIPS-only ISA feature (delay slots, `cbts`, `cbtu`, `cfromddc`,
     `ctoptr`, `creadhwr`, MIPS branch-on-tag fusion).
   - MIPS-only ABI mechanism (cap-table ABI variants, `$c13`
     varargs register, `.frame` / `.ent` directives, captable
     register `$c26`, `cheri-exact-equals` mattr).
   - MIPS-only pass / pseudo (`CheriBoundedStackPseudo*`,
     `CheriAddressingModeFolder`, `CheriRangeChecker`,
     `replaceUsesWithCheriNullReg`).
   - MIPS-only TableGen pattern (e.g. `MipsCallingConv.td` for cap-
     in-variadic; `MipsInstrCheri.td` immediate-encoding fixes).

### Biggest trap: over-erasing on MIPS-cosmetic signals

**This trap applies to CodeGen tests AND to MC parser /
operand-class / directive-semantics tests. It inverts only for MC/LLD
*reloc-encoding* tests** — there the MIPS asm/reloc *is* the subject
under test, not a cosmetic artifact (see "The MC inversion" under "Test
layers" above, incl. the sub-type split). Do not let "MC mostly erases"
suppress this trap for a parser/operand-class test. The note below is
about CodeGen, but the same logic carries to MC parser tests.

The single largest source of false-ERASE verdicts is letting
**MIPS-only flags or MIPS-asm in CHECK lines** drive the verdict to
ERASE without checking whether the *scenario* is generic.

- A crash-regression test (`-o /dev/null`, no asm CHECKs) that uses
  `-cheri-cap-table-abi=pcrel` is **still a PORT** if the crash
  itself is target-independent. The flag just selected which path
  triggered the crash on MIPS.
- A "no extra instruction" optimisation test
  (`CHECK-NOT: cincoffset` + `CHECK: clb …, 8($c3)`) is **PORT** if
  the optimisation is generic addressing-mode folding — RV does the
  same fold via its own addressing modes.
- A `ptrtoint`-in-global test that emits `.8byte ext_array` (MIPS) is
  **PORT** — the generic scenario (i64 ptrtoint of a global as a
  global initialiser) emits the per-arch directive (`.quad` on RV64,
  `.word` on RV32). The MIPS-specific directive is the artifact, not
  the scenario.
- A large-struct byval call that lowers via MIPS N64 multi-`lui`
  sequences is **PORT** if the IR scenario (passing a large struct
  byval, containing caps) is generic; the asm just happens to be
  MIPS N64.

### The two diagnostics that actually matter

1. **Where the fix lives** (Step 1, git archaeology). If the fix
   touches only files under `lib/Target/Mips/`, the bug class is
   MIPS-only by code location. If the fix touches target-independent
   code (`lib/CodeGen/SelectionDAG/`, `lib/Transforms/`, `lib/IR/`,
   `lib/MC/`, `lib/CodeGen/AsmPrinter/`, target-independent
   `TargetLowering.h`, etc.), the bug class is generic — proceed to
   the coverage check.

2. **Whether RV covers the scenario** (Step 3). Even when the bug
   class is generic, an ERASE-redundant verdict is correct if RV
   already has equivalent coverage at the same pipeline level. Cite
   the specific covering test by name and explain why it covers the
   same scenario.

## High-level flow

1. **Investigate** what the test does + git archaeology + **check the
   porting log for related tests already decided**.
2. **Classify** MIPS-specific signals vs target-agnostic signals.
3. **Check RV coverage** specifically — same scenario at same
   pipeline level.
4. **Verify empirically** — exercise the scenario on RV with the
   layer-appropriate tool (`llc` / `opt` / `llvm-mc` / `ld.lld` /
   `clang`), and **revert the fix to confirm the test still guards its
   original regression**.
5. **Present verdict + evidence** to the user. **Stop and wait for
   approval / override** before implementing.
6. **Implement** on a dedicated branch.
7. **Update the porting log** + propose a commit message.

Never proceed past step 5 without explicit user approval.

## Step 1 — Investigate the test

Read the test file in full. Identify:

- What scenario it exercises (IR shape, intrinsics, attributes,
  inline asm, RUN flags).
- What CHECK lines assert — asm pattern, structure, specific
  instructions, line ordering.
- Which RUN configurations it uses (opt level, target ABI, hybrid vs
  purecap, `-verify-machineinstrs`, etc.).
- Any FREEBSD/MIPS-only macros in the file's `target datalayout` /
  `target triple` / `RUN` comment headers.

If an inventory exists, check its preliminary verdict:

```
grep -n '<basename>' CHERI_MIPS_TEST_INVENTORY.md
```

Don't anchor on the inventory's verdict — first-pass verdicts are
often wrong (`PORT` rows re-classified to `ERASE` and vice versa).
Use it only as a starting hint.

### Git archaeology — required

For every test, find:

1. **The introduction commit** — when and why the test was added:
   ```
   git log --oneline --follow --all -- <test path>
   ```
   The earliest commit (often titled "Add test for #NNN" or similar)
   shows the bug class. The latest commits are usually mechanical
   (opaque-pointer conversion, regalloc updates, scheduler revert).

2. **The fix commit** — if the test is a bug-fix regression test, find
   the corresponding source-tree fix:
   ```
   git log --all --grep '#<issue>' --oneline
   git log --all --oneline -S '<key function>' --since='...' --until='...'
   ```
   Then `git show <hash> --stat` and inspect the diff.

The fix commit's **stat is the strongest single signal** for MIPS-
specificity. If every changed file is under `lib/Target/Mips/` (or
`MipsXxx.{cpp,h,td}` etc.), the bug class is MIPS-only by code
location.

### Check the porting log for related tests — required

Before forming a verdict, grep `CHERI_MIPS_TEST_PORTING_LOG.md` for
**already-decided tests in the same family** — same intrinsic, same
fix commit, same bug class, same source file touched, adjacent
feature. These prior decisions are the strongest available precedent
and frequently decide the verdict outright:

```bash
# By feature/intrinsic/keyword
grep -ni '<feature|intrinsic|fix-keyword>' CHERI_MIPS_TEST_PORTING_LOG.md
# By the fix commit hash or issue number
grep -ni '<fixhash>\|#<issue>' CHERI_MIPS_TEST_PORTING_LOG.md
# By the source file the fix touches
grep -ni '<SourceFile.cpp>' CHERI_MIPS_TEST_PORTING_LOG.md
```

Read the full entries for any hits. What to extract:

- **A near-identical sibling already PORTED** → the covering test now
  lives in `CHERI-Generic/` (or wherever it landed); your test is
  likely **ERASE-redundant** against *that* port. Cite it.
- **A near-identical sibling already ERASED-redundant** → the same
  covering test almost certainly covers yours too; follow the
  precedent (cite the sibling entry + the covering test it named).
  *Validated example:* `smallmemcpy.ll` (28B non-cap-multiple memcpy)
  was erased citing the earlier `lzma_index_regression.ll` (24B
  non-cap-multiple memcpy) ERASE and its named cover
  `memcpy-preserve-tags-size-not-multiple.ll`.
- **A sibling testing the *load-bearing shape* of the same fix was
  PORTED for a reason** → check whether your test produces that shape
  or a different (already-covered) one. *Example:* the overlap-into-cap
  shape was deliberately ported as
  `memcpy-webkit-overlapping-tag-cleared.ll`; a sibling whose tail is
  *not* that shape adds nothing there.
- **Coverage-discipline / empirical notes** in the sibling entry
  (e.g. "plain vs `must_preserve_cheri_tags` produced byte-identical
  asm") often transfer directly — but re-verify rather than assume.

A prior decision is precedent, not gospel — the log records its own
corrections. Still re-analyse from first principles, but treat an
unaddressed contradiction with a sibling entry as a signal you have
the verdict wrong.

## Step 2 — Classify MIPS-specificity

Concrete signals that the test exercises **MIPS-only code**:

### Strong signals (fix lives entirely in MIPS code)
- Fix files entirely under `lib/Target/Mips/`.
- Commit author note explicitly says "MIPS-specific" or "would need
  to upstream".

### Flags / passes used by the test (all MIPS-only)
- `-cheri-cap-table-abi=<plt|pcrel|fn-desc>` — MIPS cap-table ABI
  selector. The `CheriAddressingModeFolder` pass and the cap-table
  ABI machinery live only in `lib/Target/Mips/`.
- `-disable-cheri-addressing-mode-folder` — MIPS-only.
- `-cheri-cfi` — MIPS-only frame-CFI mechanism.
- `-mxcaptable=<big|small>` — MIPS-only.
- `-cheri-no-global-bounds` — MIPS-only.
- `-cheri-use-clearregs` — MIPS-only.
- `-stop-after=cheri-range-checker` — MIPS-only pass
  (`lib/Target/Mips/CheriRangeChecker.cpp`).
- `-cheri-test-mode` — (removed) MIPS-era.

### MIPS-only intrinsics / builtins
- Anything beginning `__builtin_mips_*` — MIPS-only Clang builtin.
- `int_mips_*` intrinsics defined in `IntrinsicsMips.td`.

### MIPS-only MIR pseudos
- `CheriBoundedStackPseudoImm` / `CheriBoundedStackPseudoReg` —
  defined only in `MipsInstrCheri.td` (RV uses separate `IncOffset` +
  `SetBoundsImm` MIs).

### MIPS-only directives
- `.frame`, `.ent`, `.set noreorder`, `.set nomacro`, `.set noat` —
  emitted only by MIPS `MipsTargetStreamer`. RV uses DWARF CFI
  (`.cfi_def_cfa`, `.cfi_offset`).

### MIPS-only instructions in CHECK lines
- `cfromddc`, `ctoptr`, `creadhwr`, `cgetpccincoffset`, `clcbi`,
  `cjalr`, `cjr`, `clbi`, `csbi`, `dla`, `dsll`, `daddiu`.
- `cscc`, `cllc` (MIPS atomic load-conditional).
- MIPS cap registers in CHECK lines: `$c11` (stack ptr), `$c13`
  (varargs ABI register), `$c17` (return addr), `$c18`, `$c26`
  (captable reg in PLT ABI), `$cnull`, `$ddc`. Their RV equivalents
  are completely different names.
- DDC-relative MIPS addressing: `clc $cN, $reg, 0($ddc)` — used by
  MIPS hybrid for cap load/store through int pointer.
- MIPS-only multi-word setup: `lui $1, %highest(...)`,
  `daddiu $1, $1, %higher(...)`, `dsll $1, $1, 16`, etc.

### MIPS-specific calling-conv tables
- Tests where the fix lives in `MipsCallingConv.td`.

### MIPS-only optimizations
- DAGCombines in `MipsISelLowering.cpp::performXxxCombine`.
- ISel patterns in `MipsInstrCheri.td`.
- Peepholes in `MipsSEISelDAGToDAG.cpp` (e.g.,
  `replaceUsesWithCheriNullReg`, `replaceUsesWithZeroReg`).
- Folder passes only in `lib/Target/Mips/` (e.g.,
  `CheriAddressingModeFolder.cpp`).

### MIPS-encoding bugs
- Truncated-immediate bugs in `MipsMCCodeEmitter`.
- Switch-case-missing bugs in `MipsSERegisterInfo`.

## Step 3 — Check existing RV coverage

**Run this step unconditionally — even when the bug fix is
target-independent.** A common failure mode is to short-circuit:
"fix is in `lib/CodeGen/SelectionDAG/...` → target-independent →
PORT." That conclusion is incomplete. If the same scenario is
*already* exercised on RV at the same pipeline level
(IR-fold / ISel / asm), the port adds nothing and the right call is
ERASE-redundant. Validated example: `cheri-alloca-selectiondag-regression.ll`
— the fix lives in `TargetLowering.h` (target-independent), but the
bounded-VLA-then-call scenario is already covered by
`CHERI-Generic/Inputs/bounded-allocas-lifetimes.ll::@dynamic_alloca`
(ISel-level, MIR CHECKs) AND `RISCV/cheri/stack.ll::@dynamic_alloca`
(asm-level). Verdict was ERASE-redundant, not PORT.

Decision rule: **PORT requires both (a) bug class is generic AND
(b) the scenario at the relevant pipeline level is not already
covered on RV.**

**ERASE-redundant requires positive proof, not assumption — burden is
on the ERASE.** To erase as redundant you must *exhibit* the covering
RV test **by name and by line** for every case the MIPS test pins,
including the negative/error cases (operand rejected, no-reloc emitted,
diagnostic produced). "It's probably covered", "the positive case
implies the negative", and "that case is trivial / not worth pinning"
are **not** evidence. If you grep and cannot find a specific covering
line, that absence is evidence of a *gap* → lean PORT / PORT-slim, not
ERASE. (This is the exact step we skipped on `ddc-vs-cnull.s`: an
unverified "already covered" almost drove a wrong ERASE; the grep
showed null-acceptance and `$ddc`-rejection for the cap-arith
instructions were unpinned.)

**Verify coverage claims empirically, in both directions.** Every
statement about RV coverage — "already covered" *or* "RV has no test
for this" — must be backed by an actual `grep` / `llvm-mc` / `llc` run
you performed *this session*, never recalled or inferred. This session
produced the same error twice in opposite directions (an over-claimed
gap, then an over-claimed coverage), both from asserting instead of
searching.

**A MIPS-only wrapper is not a whole-test ERASE — look for a portable
kernel (→ PORT-slim).** A test can be a thin MIPS-only shell (a
MIPS-only flag, a removed/renamed instruction, legacy-transition
diagnostics) wrapped around a target-agnostic kernel; erasing on the
shell silently discards the kernel. Before erasing a mostly-MIPS-only
test, isolate any target-agnostic behaviour it *also* pins and
coverage-check *that* — if it is a real RV gap, the verdict is
**PORT-slim** (port the kernel, drop the shell). A primitive existing
on RV (a register name, an instruction concept) makes a port *possible*
but is neither sufficient (could still be redundant) nor a blocker
(rest of the test could still be MIPS-only) — the decider is always
whether the *specific cases* are already pinned.

Be precise. Coverage analysis must match **the same scenario at the
same pipeline level** (see the existing "Coverage-verification
discipline" section of `CHERI_MIPS_TEST_PORTING_LOG.md`). Concretely:

- **Orderings / immediates / operand types** — e.g. a generic cap
  cmpxchg test that only does `acq_rel` does not cover a `seq_cst`
  cmpxchg; an integer-`xchg` test does not cover a cap-value `xchg`;
  a `ptrtoint cap to iCAPRANGE` (at-legal-width) test does NOT cover
  the sub-legal-width-promotion code path.
- **Pipeline level** — a crash/ISel regression needs an equivalent
  that exercises the same path (e.g. `--stop-after=finalize-isel` or
  `-verify-machineinstrs`), not just an asm-FileCheck of a
  superficially similar function.

### Where to look

Run targeted greps in each of these directories — coverage in **any
of them** counts as "RV-covered" for the corresponding pipeline
level:

| Directory | What it covers |
|---|---|
| `llvm/test/CodeGen/CHERI-Generic/Inputs/` | Multi-arch CHERI codegen (MIPS + RV32 + RV64 generated outputs) |
| `llvm/test/CodeGen/RISCV/cheri/` | RV-only CHERI codegen tests |
| `llvm/test/Transforms/InstCombine/CHERI/` | Target-agnostic IR-level CHERI folds |
| `llvm/test/Transforms/InstSimplify/CHERI/` | Same family |
| `llvm/test/MC/RISCV/cheri/` | RV CHERI assembler/reloc-encoding tests (MC-layer counterpart) |
| `lld/test/ELF/cheri/riscv/` | RV CHERI linker tests (cap-relocs, cap-table, ELF relocs) |
| `clang/test/CodeGen/cheri/` | Clang frontend CHERI tests |
| `clang/test/Analysis/Checkers/CHERI/` | Static analyzer (subobject-representability etc.) |

### Common grep patterns

```bash
# Find IR scenarios
grep -rln '<intrinsic or pattern>' \
  llvm/test/CodeGen/CHERI-Generic/Inputs/ \
  llvm/test/CodeGen/RISCV/cheri/ \
  llvm/test/Transforms/InstCombine/CHERI/

# Find specific bug-class fixes (where they live in source)
grep -rln '<key function or class>' llvm/lib/Target/RISCV/
grep -rln '<same>' llvm/lib/Target/Mips/

# Find RUN-line modes used
grep -E '^; RUN' <file>
```

### Tip: search both `ptr @x` and `ptr addrspace(200) @x` forms

Recurring trap: `grep 'ptrtoint @\.str'` misses
`ptrtoint (ptr addrspace(200) @x to ...)`. Widen the pattern.

## Step 4 — Empirical verification

**The empirical check is layer-dependent** (see "Test layers"):

- **CodeGen** — compile the test's IR on RV and compare asm
  structurally (the rest of this step).
- **MC** — there is no IR to compile. The decisive check is whether an
  RV counterpart already exists under `MC/RISCV/cheri/` and what it
  asserts (diff the bodies; confirm the reloc family differs —
  `R_RISCV_CHERI_*` vs `R_MIPS_CHERI_*`). For a candidate PORT, run
  `llvm-mc` on an RV-retargeted version to confirm it assembles and
  emits the expected RV reloc.
- **LLD** — assemble/link the RV-retargeted inputs and inspect the
  emitted relocs / cap-table; compare against `lld/test/ELF/cheri/riscv/`.
- **Clang** — run the frontend (`clang -cc1`) on an RV triple and
  compare emitted IR / diagnostics.

For a bug-fix regression test at any layer, the revert-the-fix check
below still applies (rebuild the relevant tool — `llc`, `llvm-mc`,
`ld.lld`, or `clang`).

### CodeGen empirical detail

Compile the test's IR on RV (not just MIPS) and **compare the asm
output structurally**, not just visually. Examples:

```bash
# RV64 purecap
build/bin/llc -mtriple=riscv64 --relocation-model=pic \
  -target-abi l64pc128d -mattr=+xcheri,+xcheripurecap,+f,+d \
  -O<level> /tmp/test.ll -o -

# RV32 purecap
build/bin/llc -mtriple=riscv32 --relocation-model=pic \
  -target-abi il32pc64f -mattr=+xcheri,+xcheripurecap,+f \
  -O<level> /tmp/test.ll -o -

# RV64 hybrid
build/bin/llc -mtriple=riscv64 --relocation-model=pic \
  -target-abi lp64d -mattr=+xcheri,+f,+d \
  -O<level> /tmp/test.ll -o -
```

(Note: the **purecap ABI** for RV32 is `il32pc64f`, not `ilp32f`;
RV64 purecap is `l64pc128d`. Getting this wrong yields
"Cannot select: RISCVISD::RET_GLUE" crashes.)

What to look for empirically:

- **Crash vs clean compile** — does RV ICE on the same IR? If MIPS
  fixed a target-independent SelectionDAG bug, RV should compile
  cleanly post-fix.
- **Structural fold or no-fold** — e.g. for chained-csetbounds:
  - MIPS post-fold: `csetbounds c18, c11+8, 40; csetbounds c3, c11+8, 40`
    (rematerialised, independent sources).
  - RV no-fold: `csetbounds a0, a0+8, 40; csetbounds s0, a0, 40`
    (chained).
- **CSV / pass-output diff** — for stats-collection tests, compare
  per-arch CSV.

When empirical and inventory disagree, **empirical wins** — but
record the surprise.

### Revert the fix — confirm the test still guards its regression

For a bug-fix regression test (Step 1 found a fix commit), the most
decisive empirical check is: **does the test actually still catch the
regression it was written for?** Temporarily revert the fix, rebuild
`llc` (or `opt`), and re-run the test's IR on **both MIPS and RV**:

```bash
# 1. Locate the live fix (it may have moved since the original commit —
#    e.g. SelectionDAG.cpp → TargetLowering.cpp after an upstream refactor).
grep -n '<guard expression / key line>' <current source file>

# 2. Neutralise it minimally (don't delete — gate it off so it's trivial
#    to restore exactly):  if (false && <guard>) { ... }
# 3. Rebuild just the tool:
ninja -C build bin/llc        # (or bin/opt)
# 4. Diff baseline vs reverted output for the test IR (and its siblings)
#    on every arch you care about:
diff <(.../llc <baseline-args> /tmp/test.ll -o -) \
     <(.../llc <baseline-args> /tmp/test.ll -o -)   # before vs after revert
```

Interpretation:

- **Output changes when the fix is reverted** → the test genuinely
  guards the regression at that pipeline level. Strong evidence the
  coverage is load-bearing → lean PORT (unless an existing RV test is
  *also* shown to change under the same revert, in which case that
  test already guards it → ERASE-redundant).
- **Output is byte-identical with the fix reverted** → the test is
  *invariant to the fix*; it no longer guards that regression on the
  arch(es) you tested. The protection is either dead/defensive code or
  comes from a different mechanism. This is a strong argument to
  **ERASE** (it asserts nothing the fix is responsible for), or, if you
  PORT, to **fix the test** so it once again pins the regression (e.g.
  pick the size/alignment/shape that actually triggers the guard).
  *Validated example:* reverting the `VT.isFatPointer()` overlap guard
  left `smallmemcpy.ll` byte-identical on MIPS **and** RV64 — proving it
  was purely a remainder-typing test (covered elsewhere), which sealed
  the ERASE-redundant verdict.

**Always restore the source exactly afterwards** (revert your `false &&`
edit) and rebuild, so the tree is clean before you present. Confirm
with `git diff --stat <file>` showing no change. This revert-rebuild is
also the right way to verify a sibling cited from the porting log
*actually* pins the fix, rather than trusting an unverified claim in a
prior entry.

## Step 5 — Present verdict to the user and stop

Show the user a structured triage that contains:

- **Bug class** with the fix commit hash + one-line summary of what
  the fix changes (and which files it touches).
- **MIPS-only signals** present (concrete list — flags, intrinsics,
  pseudos, directives, instructions in CHECK lines).
- **Related porting-log decisions** — sibling tests in the same
  family already ported/erased (from the Step 1 porting-log review),
  cited by name with how they bear on this verdict.
- **RV coverage analysis** — specific test files that do or don't
  cover the same scenario at the same pipeline level. Cite by name,
  not just by directory.
- **Empirical RV behaviour** — what RV produces on the same IR, and
  (for a bug-fix regression test) **whether reverting the fix changes
  the test's output** — i.e. whether it still guards its regression.
- **Recommendation** with one of the subcategories below.
- **Port destination** (for a PORT) — the chosen directory and why,
  plus the rejected alternative (e.g. "IR fold → `InstCombine/CHERI/`,
  not `CHERI-Generic/` which is for multi-arch asm"). See "Choosing
  the port destination" below.
- **Counter-arguments** for the opposite decision, where relevant.

### Choosing the port destination

For a PORT verdict, decide the destination from **what the test
exercises and at which pipeline level** — do not default to
`CHERI-Generic/Inputs/`. Match the destination to the test, and
state the choice (plus the rejected alternative) in the verdict so
the user can override.

| Test exercises… | Port to | Notes |
|---|---|---|
| Multi-arch codegen / asm output (the asm CHECK lines are the point) | `llvm/test/CodeGen/CHERI-Generic/Inputs/` | The 3-arch templated harness. Default *only* for genuine codegen tests. |
| A target-independent IR transform (InstCombine / InstSimplify fold, `lib/IR` / `lib/Transforms` fix) | `llvm/test/Transforms/InstCombine/CHERI/` or `.../InstSimplify/CHERI/` | Pure `opt -passes=…`, no `REQUIRES`, no MIPS asm. The fix-commit location is the tell: a `lib/IR`/`lib/Transforms` fix whose load-bearing RUN line is `opt` belongs here, next to its sibling folds. |
| RV-specific codegen behaviour with no MIPS analog worth templating | `llvm/test/CodeGen/RISCV/cheri/` | When the scenario is real but only interesting on RV. |
| Assembler / relocation encoding (MC layer, generic construct, no RV counterpart) | `llvm/test/MC/RISCV/cheri/` | Per-ISA by nature — see "The MC inversion". Most MC tests ERASE; PORT only the generic-construct gaps (e.g. `chericap-unknown-size.s`). |
| Linker behaviour (LLD layer, generic cap-reloc / cap-table concept, no RV counterpart) | `lld/test/ELF/cheri/riscv/` | Reloc-encoding half is per-ISA; linker-concept half may port. |
| Clang frontend behaviour | `clang/test/CodeGen/cheri/` or `clang/test/Analysis/Checkers/CHERI/` | Frontend / static-analyzer tests. Retarget to an RV triple. |

**Single-datalayout vs 3-arch templating.** Even when
`CHERI-Generic/` *is* the right directory, a 3-arch templated port is
not always right. If the bug only manifests at a specific capability
width — e.g. *integer-width == cap-width* collisions, where `i128`
exercises the bug on a 128-bit-cap arch but is vacuous on RV32
purecap (64-bit cap) — prefer a single representative datalayout. There
is **no `iCAPSIZE` substitution** (the table has `@CAP_BYTES@` and
`iCAPRANGE` = i64/i32/i64, but nothing for full-cap-width integers),
so templating such a test would either cover only 2/3 arches or need
`@IF-RISCV32@`-gated dual IR — both worse than one precise datalayout.
(Validated example: `invalid-instcombine-create-addrspace200.ll` →
`InstCombine/CHERI/`, single 128-bit-cap datalayout matching its
sibling `vtable-load.ll`. The fix lived in `lib/IR/Instructions.cpp`
and the load-bearing RUN line was `opt -passes=instcombine`; the MIPS
`%cheri128_llc` asm RUN line was dropped as MIPS-only cosmetic.)

Disposition subcategories we use:

| Subcategory | When |
|---|---|
| `PORT` | Standard port. Pick the destination per "Choosing the port destination" above — `CHERI-Generic/Inputs/` for codegen, `InstCombine/CHERI/` (etc.) for IR folds, `RISCV/cheri/` for RV-specific codegen. |
| `PORT-partial` | Port a subset of the original (drop MIPS-ABI-specific functions, keep the portable ones). |
| `PORT-slim` | Port a smaller representative scenario instead of the full matrix. |
| `ERASE-redundant` | Already covered by existing RV / target-agnostic tests. |
| `ERASE-MIPS-only-feature` | Tests a feature that exists only on MIPS (flags, builtins, intrinsics, directives). |
| `ERASE-MIPS-pass-local-bug` | Bug in a MIPS-only pass (e.g. `MipsSEISelDAGToDAG.cpp`). |
| `ERASE-MIPS-only-optimization` | MIPS-only optimization or DAGCombine. |
| `ERASE-MIPS-encoding-specific` | MIPS instruction- or relocation-encoding test (MC/LLD layer). Covers both encoding *bugs* and capability-*feature* assembler/reloc tests (e.g. `R_MIPS_CHERI_CAPTAB20`); RV uses a different reloc family tested under `MC/RISCV/cheri/`. |
| `ERASE-MIPS-only-directive` | Tests `.frame` / `.ent` / other MIPS-asm directive. |
| `ERASE-MIPS-only-pseudo` | Tests `CheriBoundedStackPseudo*` or similar MIPS-only MIR pseudos. |
| `ERASE-MIPS-CallingConv-specific` | MIPS calling-conv TableGen bug (`MipsCallingConv.td`). |

Then **stop**. Ask "Proceed with <recommendation>?" and wait.

## Step 6 — Implement

After approval, only:

### Branch naming

- Port: `port-<basename>`
- Erase: `remove-<basename>`
- Batched: `port-<feature-family>` or `remove-<feature-family>` (e.g.
  `remove-frame-tests`, `port-csetbounds-stats-alloc_size.ll`).

Keep the original filename (with `.ll` / `.mir` / `.c` / `.c.ll`
extension as-is). Don't rename for stylistic reasons even when the
name has historical artifacts (e.g. `cheri128.ll` stays
`cheri128.ll` even when ported to all 3 arches).

Create the branch from the base branch recorded in the porting log's
header (the branch that collects the triage work):

```
git checkout <base-branch> && git checkout -b <branch-name>
```

### For ERASE

```
git rm <test path>
```

That's it for the source tree change.

### For PORT

First settle the destination (see "Choosing the port destination" in
Step 5). The full CHERI-Generic workflow below (templating,
`regenerate-all.py`, 3-arch lit) applies **only** when the
destination is `CHERI-Generic/Inputs/`.

**For an IR-transform port** (`InstCombine/CHERI/`,
`InstSimplify/CHERI/`) the workflow is much simpler — there is no
`regenerate-all.py` and no per-arch matrix:

1. Create the file at the chosen path. Use a plain `opt -passes=…`
   RUN line (model it on a sibling like
   `InstCombine/CHERI/vtable-load.ll`: in-file `target datalayout` /
   `target triple`, no `REQUIRES`, no MIPS asm RUN line). Pick a
   single representative purecap datalayout (match a sibling's for
   directory consistency).
2. Preserve the original IR verbatim (see point 2 below).
3. Regenerate CHECK lines with
   `python3 llvm/utils/update_test_checks.py --opt-binary build/bin/opt <file>`.
   The CHECK lines (including attribute-group captures like
   `#[[ATTR0]]`) must be whatever `opt` actually emits — never
   hand-edit them to match the MIPS original; if they diverge,
   re-run the generator, don't patch by hand.
4. `build/bin/llvm-lit -v <file>`, then `git rm` the original.

Then skip to "Update the porting log". The CHERI-Generic steps below
do not apply.

**For an MC / LLD / Clang port** there is likewise no
`regenerate-all.py` and no 3-arch matrix — model the new file on its
sibling in the RV tree:

- **MC** → `llvm/test/MC/RISCV/cheri/<basename>`. Retarget the RUN
  lines to the RV `llvm-mc` invocation used by a sibling (e.g.
  `MC/RISCV/cheri/chericap-expr.s`), translate the instruction/reloc
  CHECK lines to the RV equivalents (`R_RISCV_CHERI_*`), regenerate
  with `llvm/utils/update_mc_test_checks.py` if the sibling uses it,
  and `llvm-lit -v` it.
- **LLD** → `lld/test/ELF/cheri/riscv/<basename>`, modelled on an
  existing RV sibling there.
- **Clang** → keep it in the same `clang/test/...` dir, retargeted to
  an RV triple; regenerate with `llvm/utils/update_cc_test_checks.py`
  if the original was autogenerated.

Then `git rm` the original and skip to "Update the porting log". The
CHERI-Generic steps below do not apply.

#### For PORT to CHERI-Generic

1. **Create the Inputs file.** Location:
   `llvm/test/CodeGen/CHERI-Generic/Inputs/<basename>`.

2. **Preserve the original IR verbatim** wherever it works. Recurring
   user-feedback pattern: prefer minimal diff over cleanup. Don't
   collapse trivial preheaders, don't rename SSA values, don't drop
   `; Function Attrs:` comments, don't drop unreferenced functions.

3. **Preserve original RUN flags.** Flags the user has flagged when
   we dropped them in the past:
   - `-verify-machineinstrs` (implicit in `%cheri*_llc` lit
     substitutions; must be re-added explicitly when switching to
     `llc @PURECAP_HARDFLOAT_ARGS@`).
   - `-relocation-model=static` (when the original used it; macros
     default to `pic`).
   - `-asm-verbose`, `-verify-regalloc`.
   - Opt level (`-O0` / `-O1` / `-O2` / `-O3`).
   - Multiple RUN lines (e.g. -O0 + -O3 — keep both).

4. **Preserve the original CHECK prefix** unless there's strong
   reason. Recurring user feedback: don't switch to `PURECAP` /
   `HYBRID` prefixes if the original used `CHECK`.

5. **Use templating substitutions** for cap-size-dependent values.
   The substitution engine is `regenerate-all.py`:

   | Substitution | Expands to (MIPS / RV32 / RV64) |
   |---|---|
   | `@PURECAP_HARDFLOAT_ARGS@` | per-arch purecap llc args |
   | `@PURECAP_SOFTFLOAT_ARGS@` | per-arch purecap-softfloat args |
   | `@HYBRID_HARDFLOAT_ARGS@` | per-arch hybrid llc args |
   | `@HYBRID_SOFTFLOAT_ARGS@` | per-arch hybrid-softfloat args |
   | `@CAP_BYTES@` | 16 / 8 / 16 (cap size) |
   | `@CAP_BYTES_P2@` | 4 / 3 / 4 (log2 cap size) |
   | `@CAP_RANGE_BYTES@` | 8 / 4 / 8 (address-range size) |
   | `@CAP_RANGE_BYTES_P2@` | 3 / 2 / 3 (log2 range size) |
   | `iCAPRANGE` (in IR) | i64 / i32 / i64 |
   | `@HYBRID_DATALAYOUT@` | per-arch datalayout |

6. **Use per-arch CHECK-line gating** when outputs diverge per arch:

   ```
   @IF-MIPS@; CHECK: # operand is '$c1'
   @IF-RISCV@; CHECK: # operand is '{{a0|zero}}'
   @IF-RISCV32@; CHECK: # operand is '8(sp)'
   @IF-RISCV64@; CHECK: # operand is '0(sp)'
   @IFNOT-MIPS@; CHECK: ...
   ```

   The base name `RISCV` matches both RV32 and RV64.

7. **`!DO NOT AUTOGEN!`** — add as the very first line when:
   - The test has only globals, no function bodies (autogen produces
     no CHECK lines).
   - The test verifies non-asm output (CSV stats, objdump output,
     llvm-readelf).
   - The test uses multiple RUN lines with the same FileCheck prefix
     (autogen gets confused).
   - The test is about inline-asm operand output (autogen may skip).

8. **Per-arch RUN lines** are needed when the FileCheck command takes
   per-arch arguments (e.g. `-DPTR_DIRECTIVE=...`):

   ```
   ; RUN: llc @HYBRID_HARDFLOAT_ARGS@ %s -o - | \
   @IF-MIPS@; RUN:   FileCheck %s -DPTR_DIRECTIVE=.8byte
   @IF-RISCV64@; RUN:   FileCheck %s -DPTR_DIRECTIVE=.quad
   @IF-RISCV32@; RUN:   FileCheck %s -DPTR_DIRECTIVE=.word
   ```

9. **Regenerate** the per-arch outputs:

   ```
   python3 llvm/test/CodeGen/CHERI-Generic/regenerate-all.py \
     --llvm-bindir build/bin \
     "$(git rev-parse --show-toplevel)/llvm/test/CodeGen/CHERI-Generic/Inputs/<basename>"
   ```

   The input path **must be absolute**, not relative.

   Build directory: the repo's local build at `build/bin/`. Confirm
   with `ls build/bin/llc build/bin/llvm-lit`.

10. **Verify** the generated outputs:

    ```
    build/bin/llvm-lit -v llvm/test/CodeGen/CHERI-Generic/{MIPS,RISCV32,RISCV64}/<basename>
    ```

    All three must PASS. If RV32 fails on a basic compile, double-
    check the `-target-abi` (`il32pc64f` for purecap, not `ilp32f`).

11. **Delete the original**:

    ```
    git rm llvm/test/CodeGen/Mips/cheri/<basename>
    ```

12. **Run the full CHERI-Generic suite** to confirm no regression:

    ```
    build/bin/llvm-lit -q llvm/test/CodeGen/CHERI-Generic/
    ```

### Pre-flight gotcha — `regenerate-all.py` `--opt-binary` workaround

`regenerate-all.py` passes `--opt-binary` to `update_llc_test_checks.py`,
which doesn't accept it (still the case in the CHERIoT tree as of
October 2026). If the script fails with "unrecognized arguments:
--opt-binary", either drop that argument from `llc_checks_cmd` in
`llvm/test/CodeGen/CHERI-Generic/regenerate-all.py`, or add an ignored
`--opt-binary` option to `llvm/utils/update_llc_test_checks.py`. Keep
the workaround as an **uncommitted local edit** (never include it in a
test port) and expect to re-apply it after branch switches.

## Step 7 — Update the porting log and propose a commit message

### Porting log entry

`CHERI_MIPS_TEST_PORTING_LOG.md` is the durable record. Always
update both:

1. **The Summary counts.** Find the table near the top:

   ```
   | Ported               | N  |
   | Not ported           | M  |
   | **Total decided**    | **N+M** |
   ```

   Update accordingly.

2. **A new entry under `## Decisions (chronological)`.** Insert it at
   the **top** of that section (newest first). Format:

   ```
   ### YYYY-MM-DD — `<full repo-relative path of the test>`

   **Decision:** Ported. / Not ported (erase, ...).
   Subcategory: **PORT** / **ERASE-...**.

   **Target:** (for PORT) ...

   **Branch:** `port-...` / `remove-...`.

   **Rationale (git archaeology / coverage / empirical):**
   ...

   **Verification.** (for PORT) ...

   **Templating notes.** (for PORT) ...

   **Inventory correction:** (only if an inventory exists) it had this
   as `<X>` (...); refined to **<Y>** because ...
   ```

   For ERASE entries, include the fix-commit hash + short summary +
   evidence that the fix is MIPS-only (file paths in `lib/Target/Mips/`).

3. **Side-findings** go in the `## Side findings (collect here, file
   as issues later)` section. Record factual observations
   (e.g. "RV CHERI lacks an equivalent of MIPS's CheriRangeChecker
   pass") but **do not claim bugs** if the intent (intentional design
   vs. unaddressed gap) is unclear. Frame intent-ambiguous findings
   as "factual context for future maintainers, not a bug claim."

### Commit message format

Title (under ~70 chars) — use the test's actual repo-relative path
(`test/CodeGen/Mips/cheri/…`, `test/MC/Mips/cheri/…`,
`lld/test/ELF/cheri/…`, `clang/test/…`):
- Port: `Port <path-to-test>`
- Erase: `Remove <path-to-test>`
- Batched erase: `Remove N MIPS <family> tests`
- Batched port: `Port N <family> tests`

Body — describe:
1. What the test exercised (one sentence).
2. Bug class with fix commit hash + summary.
3. Why port/erase, citing evidence (file paths, grep results,
   empirical observations).
4. Per-arch deltas (templating, dropped functions, gated CHECK
   lines).

End the body with empirical evidence (e.g. "RV64 output shows the
expected `cgettag` + mask + `sh` sequence").

Follow the target repository's convention for AI-attribution trailers
(e.g. `Co-Authored-By:`); if you don't know it, ask. The original
triage used plain commit messages by the human author.

Propose the commit message in your reply to the user; the user
commits it themselves and replies "Committed to <branch>". Don't run
`git commit` unless explicitly told to.

## Reviewer preferences learned during the original triage

- **Minimal-diff fidelity wins over cleanup.** When in tension, the
  reviewers consistently chose the verbatim form. Examples:
  - keep `; Function Attrs: inlinehint nounwind` comments even when
    inconsistent with the actual attrs;
  - keep `-relocation-model=static` even when the bug is mode-
    agnostic;
  - keep original CHECK prefixes (`CHECK`, not `PURECAP` / `HYBRID`)
    when the original used them;
  - keep multiple RUN lines with same prefix even when one would
    suffice.

- **Push back on speculative claims.** If you write "RV doesn't need
  this", be ready to prove it. Reviewers caught several over-
  reaching claims (`CheriRangeChecker` being hybrid-only; "RV doesn't
  need bounds tightening"). When evidence is incomplete, frame the
  finding as "intent unclear" rather than declaring a bug.

- **Show specific test files, not just directory names**, when
  citing RV coverage. "Covered by `RISCV/cheri/spill.ll`" is useful;
  "covered by RV CHERI tests" is not.

- **Sub-legal vs at-legal-width matters** for IR-level bug classes
  (e.g. `ptrtoint cap to i16` vs `ptrtoint cap to iCAPRANGE` exercise
  different SelectionDAG paths).

- **Keep MIPS coverage during the transition** — for target-agnostic
  features that survive MIPS removal, port even when the MIPS test
  duplicates RV-side test coverage. The future-proofing rationale is
  "preserve coverage that outlives MIPS removal."

- **For mixed batches** (some PORT + some ERASE in the same family),
  one branch is fine; branches can be split later if needed.

## Quick reference — common evidence locations

```
# Build directory
build/bin/llc                                      # llc (CodeGen)
build/bin/opt                                      # opt (IR transforms)
build/bin/llvm-mc                                  # MC / assembler tests
build/bin/ld.lld                                   # LLD / linker tests
build/bin/clang                                    # Clang frontend tests
build/bin/llvm-lit                                 # lit driver
build/bin/FileCheck                                # FileCheck

# RV-CHERI test homes (PORT destinations / coverage)
llvm/test/CodeGen/CHERI-Generic/Inputs/            # multi-arch codegen
llvm/test/CodeGen/RISCV/cheri/                     # RV-only codegen
llvm/test/MC/RISCV/cheri/                          # RV MC / reloc-encoding
lld/test/ELF/cheri/riscv/                          # RV linker tests
clang/test/CodeGen/cheri/                          # Clang frontend

# Source trees
llvm/lib/Target/Mips/                              # MIPS backend
llvm/lib/Target/RISCV/                             # RV backend
llvm/include/llvm/IR/IntrinsicsMips.td             # MIPS intrinsics
clang/include/clang/Basic/BuiltinsMips.def         # MIPS builtins
llvm/lib/Transforms/InstCombine/InstCombineCalls.cpp # InstCombine CHERI folds
llvm/lib/Transforms/Utils/CheriLogSetBoundsPass.cpp # target-indep stats pass
llvm/lib/CodeGen/AsmPrinter/AsmPrinter.cpp         # global emission
llvm/lib/CodeGen/SelectionDAG/                     # SelectionDAG

# CHERI ISA spec (authoritative for instruction/encoding-existence — see MC inversion)
# CTSRD-CHERI/cheri-specification  app-versions.tex / app-versions-8-0.tex   # per-version changelog
# UCAM-CL-TR-907 (ISAv7), UCAM-CL-TR-951 (ISAv8 = last CHERI-MIPS spec)      # cl.cam.ac.uk/techreports
```

## Things never to do

- Never run `git commit` unless explicitly told. The user commits.
- Never push to remote unless explicitly told.
- Never amend commits unless explicitly told.
- Never use `--no-verify` to bypass hooks.
- Never delete a branch or `git reset --hard` to "clean up". Discard
  uncommitted port artifacts by `rm`-ing the new files only if
  switching from PORT to ERASE for the same test (e.g. the user
  reconsidered after seeing the verdict); always recreate the branch
  from the base branch.
- Never treat a preliminary inventory verdict as authoritative —
  re-investigate every test. First-pass verdicts are often wrong.
- Never claim a test "fails on RV" without empirical proof.
- Never offer a side-finding that asserts a bug without evidence of
  *intent* — frame as "intent unclear" instead.
