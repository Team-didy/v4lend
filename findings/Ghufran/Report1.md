# [M-1] Borrowers in an infrequently-touched vault will underpay compound interest, causing lenders to under-earn

## Summary

`V4Vault._calculateGlobalInterest` accrues interest linearly over the elapsed window since the last touch: `newRate = oldRate * (1 + elapsed * rPerSec / Q64)`. If the vault is not touched for a long time at a meaningful rate, the borrower pays simple interest instead of compound. The gap versus a sequence of smaller accrual windows is `(e^(rt) − 1 − rt)`, which scales quadratically in rate × time. Under the deployed mainnet config the gap is negligible at typical touch cadence but becomes material at long gaps; under higher permitted rates it is material at short gaps. The vault has no minimum-touch mechanism and no documented assumption that the accrual window stays short.

## Root Cause

In `V4Vault.sol::_calculateGlobalInterest`:

```solidity
newDebtExchangeRateX96 = oldDebtExchangeRateX96 + oldDebtExchangeRateX96 * timeElapsed * borrowRateX64 / Q64;
newLendExchangeRateX96 = oldLendExchangeRateX96 + oldLendExchangeRateX96 * timeElapsed * supplyRateX64 / Q64;
```

The formula is a single-shot linear accrual. There is no compounding within the window, no maximum-window cap, and no documented requirement that the vault be touched frequently. Compare to Compound V2, which compounds on every `accrueInterest()` call (per block); the linear-over-gap approximation there is bounded by a daily accrual period, not an arbitrary one.

## Internal Pre-conditions

- `V4Vault.interestRateModel` is configured with non-zero rates. The deployed mainnet config (`base = 0%`, `multiplier = 5%`, `jumpMultiplier = 109%`, `kink = 80%`) satisfies this at any utilization above zero.

- Vault utilization is at or above the kink for the duration of the gap. At the deployed config this yields ~26% APY; below kink the effect is proportionally smaller.

- No state-changing function on V4Vault is called for the duration of the gap (deposit, withdraw, borrow, repay, transform, create, remove, setReserveFactor, setLimits, liquidate, decreaseLiquidityAndCollect, transferLoan, approveTransform). _updateGlobalInterest runs on all of them, so any of these ends the linear window.

- `block.timestamp > lastExchangeRateUpdate` at the moment of the next accrual.

## External Pre-conditions

None. The condition is entirely endogenous to the vault.

## Attack Path

This is a vulnerability path, not an active attacker path. No external actor is required; the leak is structural.

- A borrower opens a loan against a vault-held position.

- Vault utilization rises above the kink and no state-changing call reaches V4Vault for the duration of the gap.

- On the next touch, _calculateGlobalInterest computes the new exchange rates using oldRate × (1 + r·t) — the linear form.

- The borrower repays at the under-charged rate. Lenders redeem at the under-charged rate. Reserves are under-accrued.

- The gap between the linear value and the true compounded value is permanently lost, because the next accrual window starts from the linear rate.

### Impact

Lenders and the protocol reserve share are under-paid — no principal loss, but the borrower keeps the difference — with a magnitude ranging from sub-1bp at typical per-block/per-day touch cadence to ~5–8% of initial debt at the deployed config after 5 years untouched and ~1% at the max permitted config after 30 days untouched, the gap arising from the single-shot linear formula in _calculateGlobalInterest (missing compounding plus utilization frozen at the window's starting value); eliminated by per-touch accrual or per-window compound-plus-recompute.
 
## Fix

- Replace the linear formula with a compound accrual over the elapsed window. Any of:

Correct exponential approximation.

Compound-per-day: `newRate = oldRate × (1 + rPerDay)^(days) × (1 + r × remainder)`, which caps the linear error to a single day.

Or `newRate = oldRate × (1 + r)^elapsed` using `r = per-second` truncated to a compoundable form.

## POC



