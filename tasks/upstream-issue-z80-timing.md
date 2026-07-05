# Draft: upstream issue for davidly/ntvcm

Status: DRAFT, not yet filed. Needs explicit go-ahead before posting to
davidly/ntvcm (per the "explain root cause + get explicit per-filing
go-ahead before any upstream post" rule for third-party repos).

Target: https://github.com/davidly/ntvcm/issues (upstream of this fork,
ravn/ntvcm). Fixed here in ravn/ntvcm commit 5bc76e5 (tests in 22fb397);
not yet proposed upstream.

---

## Title

Z80 `-p` cycle counter undercounts T-states by ~30% on typical code
(DJNZ/JR, conditional JP/CALL, IX/IY indexed memory)

## Summary

Comparing `ntvcm -p`'s reported Z80 cycle count against z88dk's `ticks`
(an independent cycle-accurate Z80 emulator, https://github.com/z88dk/z88dk,
`src/ticks`) on the same compiled `.com` binary shows ntvcm undercounting
by roughly 30% on a typical C program (a Sieve of Eratosthenes compiled
with `dcc`). Program *output* is always correct -- this is purely a
cycle-counting bug in the Z80 timing model, not a correctness bug.

Root-caused to three separate issues in `x80.cxx`'s `z80_emulate()`, all
in the same family: for opcodes where `z80_cycles[op]`/`i8080_cycles[op]`
(the shared base-cost table indexed in the main dispatch loop) is 0, the
`cycles` value assigned inside `z80_emulate()`'s switch is meant to be
the *complete* T-state cost for that opcode (since the base contributes
nothing) -- but several of these assignments look like leftover
M-cycle-count placeholders rather than real T-state counts.

## Bug 1: DJNZ / JR / JR cc use M-cycle counts instead of T-states

`case 0x10` (DJNZ) and `case 0x18/0x20/0x28/0x30/0x38` (JR, JR NZ/Z/NC/C)
use `cycles = 2` or `cycles = 3` for both taken and not-taken paths.
Real Z80 timing: DJNZ is 13 T taken / 8 T not-taken; JR is 12 T
unconditional; JR cc is 12 T taken / 7 T not-taken.

### Repro

5-byte CP/M `.com`, org 0x100:

```
0100  06 00        LD   B, 0
0102  10 fe        DJNZ 0102          ; loop: jump to self
0104  c9           RET
```

B starts at 0, so DJNZ executes 256 times: 255 taken + 1 not-taken.
Hand-calculated expected cost: `7 + 255*13 + 1*8 + 10 = 3340` T
(`LD B,0` + the 256 DJNZs + `RET`).

Cross-checked against z88dk `ticks` (raw CPU emulator, no CP/M layer)
loading the same 5 bytes at address 0 and running with `-pc 100 -end 0`
(with the first 2 bytes of the image zeroed so the final bare `RET`
pops a real 0x0000 off the stack): ticks reports exactly **3340**,
matching the hand calculation.

`ntvcm -p` on the same 5 bytes (as a `.com`, run under its own CP/M
layer): reports **808** before the fix. ntvcm's number can't be
directly compared byte-for-byte to ticks' 3340 because ntvcm, being a
real CP/M implementation, executes a short warm-boot `JP` chain after
this program's `RET`-to-0x0000 that ticks' minimal harness never
executes (empirically a constant +24 T across every test in this
report) -- but 808 is nowhere near 3340+24=3364.

### Fix

Use the real T-state values: `13/8` for DJNZ, `12` for JR, `12/7` for JR
cc. After the fix, ntvcm reports exactly 3364 for this repro.

## Bug 2: conditional JP / CALL over-subtract on the not-taken path

A shared constant (`cyclesnt`, value 6 in this codebase) is subtracted
from the "taken" cost to get the "not-taken" cost, for conditional RET,
JP, and CALL alike. That's only correct for conditional RET (Z80/8080:
11 T taken, 5 T not-taken, difference 6). It's wrong for the other two:

- `JP cc` costs 10 T whether taken or not -- there should be no
  subtraction at all.
- `CALL cc` differs by ISA: 8080 not-taken saves 6 T (11 vs 17,
  correct, matches `cyclesnt`), but Z80 not-taken saves 7 T (10 vs 17).

### Repro

9-byte CP/M `.com`, org 0x100:

```
0100  06 00        LD   B, 0
0102  af           XOR  A
0103  c2 06 01     JP   NZ, 0106      ; never taken (Z flag set by XOR A)
0106  10 fa        DJNZ 0102
0108  c9           RET
```

256 iterations of a *never-taken* `JP NZ` plus a DJNZ (255 taken + 1
not-taken) to close the loop. Expected: `7 + 256*(4+10) + 255*13 + 1*8 +
10 = 6924` T. z88dk `ticks` on the same bytes (same harness as above):
exactly **6924**.

`ntvcm -p`, with bug 1 already fixed but bug 2 not yet fixed, reports
**5412** -- a deficit of exactly `256 * 6 = 1536`, i.e. precisely the
number of not-taken `JP NZ` occurrences times the erroneous `cyclesnt`
subtraction. (With neither bug fixed, i.e. current released ntvcm,
it reports 2856, combining both deficits.)

### Fix

Don't subtract anything for conditional JP not-taken. For conditional
CALL not-taken, subtract 7 in Z80 mode (`cyclesnt` unchanged at 6 for
8080 mode). After both fixes, this repro reports exactly 6948 (6924 +
the same +24 warm-boot constant as bug 1).

## Bug 3: IX/IY indexed memory access uses the same placeholder pattern

Same class of bug as #1: `LD r,(IX+d)`/`LD (IX+d),r`/`LD (IX+d),n`/math
on `(IX+d)` use `cycles = 5`; `INC (IX+d)`/`DEC (IX+d)` use `cycles = 6`.
Real Z80 timings are 19 T and 23 T respectively.

This is the dominant contributor on realistic compiled C: compilers
(dcc, sdcc) commonly use IX as a frame pointer, so `ld r,(ix+d)`-style
local-variable access is by far the hottest opcode pattern in real
programs -- ntvcm's own `-g` per-PC profiling flag showed 13,413 and
13,389 executions of two such instructions alone in a modest Sieve of
Eratosthenes benchmark.

### Repro / measurement

Rather than another isolated micro-repro, this one was measured on a
real compiled program end-to-end, since IX-indexed access is inherently
tied to a compiler's calling convention and isn't as easy to isolate by
hand as bugs 1-2: `dcc`-compiled Sieve of Eratosthenes (`.com`, 1408
bytes; see rc700-gensmedet/tasks/compiler-comparison-corpus in the
ravn/z80-compiler-suite-workspace repo, `build_dcc_corpus.sh sieve
size`). z88dk `ticks` on the equivalent raw memory image (with a minimal
hand-built warm-boot + BDOS-function-0 stub, `-pc 100 -end 0`): **9,838,550**
T-states. Program correctness confirmed independently (1007 primes
found, matching the expected result) via ntvcm's memory-dump output,
so the discrepancy is purely a timing issue.

| stage                                    | ntvcm -p result | vs ticks (9,838,550) |
|-------------------------------------------|-----------------:|----------------------:|
| before any fix                             |         6,742,803 |                 -31.5% |
| after bug 1 fixed only                     |         6,742,831 |                 -31.5% |
| after bugs 1+2 fixed                       |         6,862,805 |                 -30.3% |
| after bugs 1+2+3 fixed                     |         9,710,500 |                  -1.3% |

A second, independently-built benchmark (fannkuch, dcc, size-optimized)
shows the same pattern: ticks 102,632,291 T vs ntvcm 102,432,304 T after
all three fixes (-0.2%).

### Fix

Use 19 T for `LD r,(IX+d)` / `LD (IX+d),r` / `LD (IX+d),n` / math on
`(IX+d)`, and 23 T for `INC (IX+d)` / `DEC (IX+d)`.

## Residual gap (open question, not part of this issue's fix)

After all three fixes, ntvcm still undercounts by ~0.2-1.5% on the two
real-program benchmarks above (sieve, fannkuch). This has not been
root-caused. The most likely remaining suspect is the CB-prefixed
IX/IY-indexed bit/rotate group in the same file, which has at least one
`cycles = 8` value explicitly commented "this is a guess -- it's
undocumented" -- but this has NOT been verified with an isolated repro
the way bugs 1-3 were, so it should not be asserted as fact in the
upstream report. If filed, this should be flagged as a known, separate,
unconfirmed residual, not folded into the fix.

## Suggested fix (as already applied in ravn/ntvcm)

See ravn/ntvcm commit 5bc76e5 for the full diff, and commit 22fb397 for
a small regression-test harness (`tests/timing/`, three of the four
repros above as `.com` files plus a script + hand-derived expected
values) that was red/green validated against the pre-fix commit.

## Before filing

- [ ] Confirm repro programs still exactly reproduce on a clean checkout
      of davidly/ntvcm main (not just the ravn fork) -- has NOT been
      done yet, only tested against the ravn fork's tree.
- [ ] Decide whether to submit as an issue only, or issue + PR (fork
      already has the fix + tests ready to cherry-pick/rebase).
- [ ] Get explicit go-ahead per the upstream-filing diligence rule
      before posting.
