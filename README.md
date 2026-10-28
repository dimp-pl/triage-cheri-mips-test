# triage-cheri-mips-test

A [Claude Code](https://claude.com/claude-code) skill for deciding, one test at a time, whether a CHERI-MIPS test in CHERI LLVM should be **ported** to RISC-V CHERI or **removed**, with evidence for every decision.

It comes from the talk *"Do We Even Need This Test? What to do with a retiring backend's decade of tests"* at the 2026 US LLVM Developers' Meeting.

It's a written procedure, not a tool: the agent collects the evidence and stops, and a human makes every decision.

## What it does for each test

1. **Investigate.** Read the test, find the commit that added it and the fix it guards (git archaeology), and check earlier decisions on related tests.
2. **Classify.** Look for MIPS-only signals. The strongest one is a fix that lives entirely in `lib/Target/Mips/`.
3. **Check RISC-V coverage.** The cover must exercise the same scenario at the same pipeline level, named by file and line. A file with a similar name doesn't count.
4. **Verify by running things.** Compile for RV32 and RV64. For bug-fix tests, switch the fix off, rebuild, and see which tests trip.
5. **Present a verdict in a fixed shape, then stop.** The verdict covers the fix commit, MIPS-only signals, related decisions, RISC-V coverage, empirical results, the recommendation and destination, and counter-arguments. It ends with *"Proceed with &lt;recommendation&gt;?"*.
6. **Implement** on a `port-…` or `remove-…` branch, in the right destination: `CHERI-Generic/Inputs`, `RISCV/cheri`, `MC/RISCV/cheri`, lld, clang or `InstCombine/CHERI`.
7. **Record** the decision in the porting log and propose a commit message. You commit.

## Install

Clone this repository, then copy the skill folder into an llvm-project checkout from the CHERI LLVM family, for example [CHERIoT-Platform/llvm-project](https://github.com/CHERIoT-Platform/llvm-project):

```
git clone https://github.com/dimp-pl/triage-cheri-mips-test.git
mkdir -p <llvm-checkout>/.claude/skills
cp -r triage-cheri-mips-test/triage-cheri-mips-test <llvm-checkout>/.claude/skills/
```

To make it available in every project, copy it to `~/.claude/skills/` instead.

## Use

```
/triage-cheri-mips-test llvm/test/CodeGen/Mips/cheri/<test>
```

Asking in plain words also works: *"triage llvm/test/MC/Mips/cheri/&lt;test&gt;"*.

## Prerequisites

- **Full git history.** The archaeology step needs it.
- **A build in `build/`** with `llc`, `opt`, `llvm-mc`, `ld.lld`, `clang`, `llvm-lit` and `FileCheck`. Enable assertions if you want crash regressions to show up. The revert experiment rebuilds single tools, e.g. `ninja -C build bin/llc`.
- **Python 3**, for `llvm/test/CodeGen/CHERI-Generic/regenerate-all.py`. In the CHERIoT tree it currently needs a small local workaround, described in the skill under "Pre-flight gotcha".

## Working files

The skill keeps two files in the root of the checkout, created from `triage-cheri-mips-test/templates/` the first time they're needed:

- **`CHERI_MIPS_TEST_PORTING_LOG.md`** (required). The decision log, newest first, with the policy sections the skill relies on: minimal-diff porting, coverage-verification rules, and erase categories. On first use the skill asks which branch collects the work.
- **`CHERI_MIPS_TEST_INVENTORY.md`** (optional). A first-pass list of all tests with *preliminary* verdicts, created only if you ask for it. Treat it as a to-do list: first-pass verdicts are often wrong.

Keep both untracked, e.g. by adding them to `.git/info/exclude`.

## Using it for another backend

Most of the procedure isn't specific to CHERI or MIPS:

| Reusable as is | Replace for your project |
|---|---|
| The seven-step flow and the STOP before deciding | "Test layers" and where ports go |
| Git archaeology: where does the fix live? | MIPS-only signal lists (Step 2) |
| Coverage-verification discipline | `llc` flags for RV32/RV64 (Step 4) |
| Revert the fix, see what trips | CHERI-Generic templating (Step 6) |
| The fixed verdict template | "Consult the CHERI ISA spec" |
| The decision log and "Things never to do" | Quick-reference paths |

A cheap way to start without any AI: for each test, find the commit that added it and check whether its code change touched only `lib/Target/<Backend>/`. In the original run, 85 of the 97 tests whose adding commit changed only MIPS backend code were removed. For tests whose adding commit changed only target-independent code, the outcome was close to a coin flip, decided by whether RISC-V already had an equivalent test. In upstream LLVM, tests are often committed separately before the fix (see the TestingGuide), so look for the commit that changed the test's CHECK lines together with code.

## The original run

All 206 CHERI-MIPS CodeGen and MC tests in CHERIoT LLVM were triaged with this procedure, which became a skill partway through the work. 53 were ported and 153 removed, in 151 patches, all of which landed in [CHERIoT-Platform/llvm-project](https://github.com/CHERIoT-Platform/llvm-project). The CHERI-Generic suite grew from 40 to 80 inputs. The skill's clang and lld sections are written but haven't been exercised at that scale yet.

## Credits

Written by Zmicier Prybysh for BayLibre.

## License

Apache License v2.0 with LLVM Exceptions (`Apache-2.0 WITH LLVM-exception`). See [LICENSE](LICENSE).
