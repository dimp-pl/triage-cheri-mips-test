# CHERI MIPS Test Inventory

One row per MIPS-CHERI test, with a **preliminary** verdict from a quick
first pass. It's a to-do list, not a verdict: every test is re-triaged
from first principles before anything is ported or removed, and the
final decision is recorded here and in the porting log. Keep this file
untracked in the llvm-project checkout.

To fill it in, list the tests in scope, for example:

```
git ls-files llvm/test/CodeGen/Mips/cheri llvm/test/MC/Mips/cheri
git grep -l -i mips -- lld/test/ELF/cheri clang/test | grep -i cheri
```

Judge the first pass by **what the test exercises** (IR or source,
RUN flags, the pass or path under test), not by the MIPS mnemonics in
its CHECK lines: those only show which backend ran.

Preliminary verdicts: `PORT`, `ERASE` (MIPS-only), `ERASE-REDUNDANT`
(already covered on RISC-V), `AMBIGUOUS`.

## `llvm/test/CodeGen/Mips/cheri/`

| Test | What it exercises | Preliminary | Why | Final | Log entry |
|---|---|---|---|---|---|

## `llvm/test/MC/Mips/cheri/`

| Test | What it exercises | Preliminary | Why | Final | Log entry |
|---|---|---|---|---|---|

## `lld/test/ELF/cheri/` (MIPS-bound)

| Test | What it exercises | Preliminary | Why | Final | Log entry |
|---|---|---|---|---|---|

## `clang/test/**/cheri/` (MIPS-bound)

| Test | What it exercises | Preliminary | Why | Final | Log entry |
|---|---|---|---|---|---|
