# [AUTOMATED] ARM LDRH return inferred as signed short changes caller arithmetic

Kuna infers a signed `int2` result for an ARM function that returns a
zero-extending `LDRH`. The emitted caller then sign-extends that result through
C integer promotion and computes the wrong value. An unsigned-short prototype
for the same callee restores the correct result on the same instructions.

## Minimal asset-free reproduction

Download [kuna-arm-ldrh-return-repro.tar.gz](https://github.com/mgostIH/kuna/raw/refs/heads/repro/arm-ldrh-return-20261002/repros/arm-ldrh-return/kuna-arm-ldrh-return-repro.tar.gz),
extract it, then run:

```sh
cd kuna-arm-ldrh-return-repro
python3 repro.py --kuna /absolute/path/to/kuna --sleighpath /absolute/path/to/specs
```

The archive contains a Python standard-library ELF builder/runner, the 676-byte
ELF, documented assembly, assertions, captured JSON/C, execution results and
file hashes. It needs no compiler, assembler, game binary, game declarations or
assets to reproduce decompilation. The independently authored `.text` contains
52 bytes, including one four-byte literal pointer; writable state is two bytes.

Both cases run serially at nice +15 and idle I/O priority, in reliable mode,
ARM state, with strict assertions. All four baseline assertions and all five
control assertions apply. Both CLI exits are zero and function/global error
fields are null. The runner returns zero when the reported defect and passing
control reproduce; a fixed version will fail its expected-defect check.

Optional actual ARM execution and compiled-C comparison, with Unicorn 2.1.4
and GCC installed:

```sh
uv run --with unicorn==2.1.4 python repro.py \
  --kuna /absolute/path/to/kuna --sleighpath /absolute/path/to/specs --verify
```

## Expected behavior

`Next`, at `0x10000`, increments a writable halfword and reloads it with `LDRH`:

```asm
ldr  r1, [pc, #16]  // literal at 0x10018 contains 0x11000
ldrh r0, [r1]
add  r0, r0, #1
strh r0, [r1]
ldrh r0, [r1]
bx   lr
```

`Scale`, at `0x1001c`, preserves its incoming bound across `BL Next`, multiplies
the full r0 result by that bound, and uses `ASR #16`. Starting from state
`0x7fff`, `Next` returns r0=`0x00008000` and leaves state=`0x8000`.
With bound 100, the product is 3276800 and `Scale` returns **50**.

The expected emission must preserve that zero-extension: an unsigned narrow
return or a full-width result consistent with the instruction would work.

## Actual emission and control

Baseline output:

```c
int2 Next(void)
{
  state16 += 1;
  return state16;
}

int4 Scale(int4 bound)
{
  return bound * Next() >> 0x10;
}
```

The signed 16-bit return promotes `0x8000` to -32768. Multiplication by 100
and arithmetic shift yield **-50**. There is no signed multiplication overflow
in this counterexample. GCC/Linux compilation of these exact bodies, with the
sized types declared and `state16` as unsigned short, confirms the wrong value.
Unicorn execution of the authored ARM instructions confirms **50**.

The control adds only:

```text
prototype Next unsigned short Next(void)
```

It emits `uint2 Next(void)` and:

```c
return (int4)(bound * (uint4)Next()) >> 0x10;
```

The optional check reports:

```text
Actual ARM: result=50, state=0x8000
baseline compiled C: result=-50, state=0x8000
unsigned compiled C: result=50, state=0x8000
```

This establishes an observable signedness error across a call, rather than a
type-spelling preference. The report does not prescribe an unverified internal
fix or investigate register/stack lifetime recovery.

## Version and duplicate review

Verified on a clean source build of official `main`
`5dd534d8df3f26c9e2c972f12610cbc7773d1d44`. GitHub's `commits/main` endpoint
still resolved to that revision when checked on 2026-10-02. The checkout has no
source modifications. This build's CLI reports `kuna 0.1.0`, so the commit and
binary hash are the identifiers used here:

- Kuna SHA-256: `767070457f2cd0066fb7033a2622912b9faaca70d668b6bd21e45df5b09c7ae9`.
- Fixture SHA-256: `0d806ad5cc491169082c0367dcde8286f74b58726ccc14e6793cde59f694cf22`.
- Target: `ARM:LE:32:v5t:default`, `--isa arm`, `--mode reliable`, `--jobs 1`.

Targeted open/closed issue searches for `LDRH` and `halfword` found no matches.
Searching issues and PRs together for `LDRH` found only the unrelated literal
pool/string work in [PR #431](https://github.com/Noelo-Lab/kuna/pull/431).
The broader unsigned-return results were screened and the closest records
were read directly:

- [#769](https://github.com/Noelo-Lab/kuna/issues/769) concerns a zero-extended
  64-bit result narrowed to signed 32-bit, rather than this ordinary 16-bit
  memory reload and caller promotion.
- [#816](https://github.com/Noelo-Lab/kuna/issues/816) concerns extension of
  explicitly declared narrow returns on MIPS/RISC-V. This fixture instead
  exhibits an inferred ARM return type; its explicit unsigned control works.

No exact duplicate was found in those searches. The signedness mechanism is
related to #769, so this reduction may be useful as an additional narrow-width
regression. Search terminology can miss an existing report.

Discovered during game reverse engineering; the report and archive contain
only independently authored synthetic inputs and no game content.
