
# Uncaught Integer Overflow During AbstractInstructionSet Constant Folding

Submitted on Wed Jun 26 2024 06:02:05 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32548

Report type: Smart Contract

Report severity: Low

Target: https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59

Impacts:
- Incorrect sway optimization leading to incorrect bytecode

## Description
## Brief/Intro

`const_indexing_aggregates_function` multiplies Constants without checking whether the result overflows. This might incorrectly optimize away overflowing operations.

## Vulnerability Details

`const_indexing_aggregates_function()` constant folds the `MUL` instruction if both operands are Constant values. However, it does not check whether the result overflows, and can incorrectly optimize overflowing multiplications.

```
VirtualOp::MUL(dest, opd1, opd2) => {
    match (reg_contents.get(opd1), reg_contents.get(opd2)) {
        (Some(RegContents::Constant(c1)), Some(RegContents::Constant(c2))) => {
            reg_contents.insert(dest.clone(), RegContents::Constant(c1 * c2));
            record_new_def(&mut latest_version, dest);
        }
        _ => {
            reg_contents.remove(dest);
            record_new_def(&mut latest_version, dest);
        }
    }
}
```

Compilation of the code would replace the load with a direct read from `hp`, where all the overflowing intermediate calculations on `a` are constant folded away.

```
fn constant_folding_overflow() -> u64 {
    let a = asm(a, b, c) {
        movi a i16;
        aloc a;
        move c hp;
        addi b c i0;    // base_reg = c, offset = 0
        movi a i65536;  // a = 1 << 16
        mul a a a;  // a = a * a = 1 << 32
        mul a a a;  // a = a * a = 1 << 64 = 0
        add b b a;  // base_reg = c, offset = 0
        lw a b i0;
        a: u64
    };
    a
}
```

```
.program:
.2                                      ; --- start of function: constant_folding_overflow ---
move $$locbase $sp                      ; save locals base register for constant_folding_overflow
cfei i8                                 ; allocate 8 bytes for locals and 0 slots for call arguments.
.5
movi $r0 i16                            ; movi a i16
aloc $r0                                ; aloc a
move $r0 $hp                            ; move c hp
lw   $r0 $r0 i0                         ; lw a b i0
sw   $$locbase $r0 i0                   ; store value
lw   $r0 $$locbase i0                   ; load value
ret  $r0
```

## Impact Details

As usual, it is hard to come up with a precise impact estimation of incorrect code generation because it depends on what code the user writes. The best case scenario would be contracts that run into those bugs getting bricked, and the worst case scenario would be that incorrect program behaviors lead to loss of funds.

## References

- `https://github.com/FuelLabs/sway/blob/2cbc24dc2e4ecab1e2b65fb8542d4650e313db99/sway-core/src/asm_generation/fuel/optimizations.rs#L115`
        
## Proof of concept
## Proof of Concept

This test would fail because the overflowing multiplications on `a` is constant folded away and not included in the generated bytecode.

The PoC should be run with release build of `forc` because rust debug build includes arithmetic overflow checks.

```
#[test(should_revert)]
fn constant_folding_overflow() -> u64 {
    let a = asm(a, b, c) {
        movi a i16;
        aloc a;
        move c hp;
        addi b c i0;	// base_reg = c, offset = 0
        movi a i65536;	// a = 1 << 16
        mul a a a;	// a = a * a = 1 << 32
        mul a a a;	// a = a * a = 1 << 64 = 0
        add b b a;	// base_reg = c, offset = 0
        lw a b i0;
        a: u64
    };
    a
}
```