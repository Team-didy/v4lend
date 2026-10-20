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

Please add below test in `test/poc/PoC_S1F1_SimpleInterest.t.sol` and run `forge test --match-contract PoC_S1F1_SimpleInterest -vvv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.30;

import {console} from "forge-std/console.sol";
import {V4VaultLocalBase} from "test/vault/support/V4VaultLocalBase.sol";
import {InterestRateModel} from "src/vault/InterestRateModel.sol";

/// @title PoC — S1-F1: Simple interest under-compounds over long inactive periods
///
/// @notice FINDING
///   `V4Vault._calculateGlobalInterest` accrues linearly within a single window:
///       newRate = oldRate * (1 + elapsed * rPerSec / Q64)
///   It does not compound, has no per-window cap, and no documented requirement
///   that the vault be touched frequently. The gap versus a sequence of smaller
///   accrual windows is (e^(rt) - 1 - rt), which scales quadratically in r * t.
///
/// @notice REGRESSION MARKER
///   Both tests PASS while the bug is present and FAIL once
///   `_calculateGlobalInterest` compounds correctly. When fixed:
///     - Test 1: `simple == compound` (within dust) under the deployed config.
///     - Test 2: `simple == compound` (within dust) under the max config.
///   Update the assertion bodies accordingly.
///
/// @dev Inherits `V4VaultLocalBase`: fork-free, real PoolManager/PositionManager,
///      mock oracle, mock asset, zero-rate InterestRateModel by default. This PoC
///      overrides the rate model per test so the simple-vs-compound gap is
///      observable within a practical runtime.
contract PoC_S1F1_SimpleInterest is V4VaultLocalBase {
    uint256 constant Q64 = 2 ** 64;

    // ---------------------------------------------------------------- configs

    /// @dev Matches `script/DeployMainnet.s.sol`:
    ///      base = 0%, multiplier = 5%, jump = 109%, kink = 80%.
    function _installDeployedRate() internal {
        InterestRateModel(address(vault.interestRateModel())).setValues(
            0,
            Q64 * 5 / 100,
            Q64 * 109 / 100,
            Q64 * 80 / 100
        );
    }

    /// @dev Max permitted by `setValues`:
    ///      base = 10%, multiplier = 200%, jump = 200%, kink = 0.
    ///      NOT the deployed config; demonstrates the mechanism at a short horizon.
    function _installMaxRate() internal {
        InterestRateModel(address(vault.interestRateModel())).setValues(
            Q64 / 10, Q64 * 2, Q64 * 2, 0
        );
    }

    // ---------------------------------------------------------------- helper

    /// @dev Returns (simple, compound, initial) for one untouched window vs
    ///      daily-touched windows over the same total wall-clock time.
    ///      Uses `vm.snapshot` / `vm.revertTo` so both paths start from the same
    ///      post-borrow state.
    function _measureGap(uint256 daysElapsed)
        internal
        returns (uint256 debtSimple, uint256 debtCompound, uint256 debtInitial)
    {
        // Stage: deposit, mint collateral, borrow to 80% util against the mock value.
        _deposit(lender, 1000e18);
        oracle.setMockPositionValue(1000e18);
        uint256 tokenId = _createLoan(uint128(1000e18));
        _borrow(tokenId, 800e18);
        (debtInitial,,,,) = vault.loanInfo(tokenId);

        uint32 rf = vault.reserveFactorX32();
        uint256 startTs = block.timestamp;
        uint256 snap = vm.snapshot();

        // A) untouched: one accrual covers the whole window, linear formula.
        vm.warp(startTs + daysElapsed * 1 days);
        vault.setReserveFactor(rf); // any state-changing call triggers _updateGlobalInterest
        (debtSimple,,,,) = vault.loanInfo(tokenId);

        // B) compounded: same wall-clock, one accrual per day.
        vm.revertTo(snap);
        vm.warp(startTs); // safety net: some forge versions need this after revertTo
        for (uint256 i = 0; i < daysElapsed; i++) {
            vm.warp(startTs + (i + 1) * 1 days);
            vault.setReserveFactor(rf);
        }
        (debtCompound,,,,) = vault.loanInfo(tokenId);
    }

    // ---------------------------------------------------------------- test 1

    /// @notice Deployed mainnet config, 5 years untouched.
    /// @dev Passes while the bug is present. At the deployed config this is the
    ///      minimum horizon where the linear-vs-compound gap is material.
    function test_RegressionMarker_DeployedConfig_LongGap() public {
        _installDeployedRate();

        (uint256 simple, uint256 compound, uint256 initial) = _measureGap(5 * 365);

        console.log("== deployed config, 5 years untouched ==");
        console.log("Initial debt:    ", initial);
        console.log("Simple  (buggy): ", simple);
        console.log("Compound expect: ", compound);
        console.log("Gap absolute:    ", compound - simple);
        console.log("Gap % of debt:   ", (compound - simple) * 100 / initial);

        // REGRESSION MARKER: bug present -> simple < compound. Fix -> equal within dust.
        assertLt(simple, compound, "BUG: untouched accrual under-charges vs compounding");
        // Materiality: gap is >1% of initial debt under the deployed config at 5y untouched.
        assertGt(compound - simple, initial / 100, "BUG: gap is >1% of debt at 5y untouched");
    }

    // ---------------------------------------------------------------- test 2

    /// @notice Max permitted config, 30 days untouched.
    /// @dev Passes while the bug is present. Demonstrates the mechanism without
    ///      requiring the long horizon test 1 needs under the deployed config.
    function test_RegressionMarker_MaxConfig_MechanismAt30Days() public {
        _installMaxRate();

        (uint256 simple, uint256 compound, uint256 initial) = _measureGap(30);

        console.log("== max config, 30 days untouched ==");
        console.log("Initial debt:    ", initial);
        console.log("Simple  (buggy): ", simple);
        console.log("Compound expect: ", compound);
        console.log("Gap absolute:    ", compound - simple);

        // REGRESSION MARKER: bug present -> simple < compound. Fix -> equal within dust.
        assertLt(simple, compound, "BUG: linear accrual under-charges vs compounding");
        // Materiality under max config at 30 days.
        assertGt(compound - simple, initial / 1000, "BUG: gap >0.1% of debt at max config");
    }
}
```

