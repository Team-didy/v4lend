## [M-2] A passive dust-underwater position will keep accruing phantom debt against lenders, and the eventual realization haircuts every remaining lender in a single step

### Summary

When a loan reaches `fullValue ≤ debt × 10%`, `_calculateLiquidation` returns `liquidatorCost = 0` and `liquidationValue = fullValue`. The collateral reward is dust; gas exceeds it; no rational liquidator calls liquidate. The vault exposes no writeOffLoan, sweepBadDebt, forceLiquidate, or governance-triggered cleanup, and no minimum-keeper-reward path. The phantom debt remains in `loans[tokenId]` and debtSharesTotal indefinitely, keeps accruing interest in `_calculateGlobalInterest`, and inflates lendExchangeRateX96 for the entire interval. When realization eventually occurs, _handleReserveLiquidation writes the accumulated shortfall down in a single step; every lender present at that instant absorbs it at once. Depositors who joined during the phantom accrual were over-charged for their shares; depositors who withdrew before realization were over-paid.

### Root Cause

In V4Vault.sol::_calculateLiquidation

```solidity
if (fullValue > penalty) {
    liquidatorCost = fullValue - penalty;
} else {
    // this extreme case leads to free liquidation
    liquidatorCost = 0;
}
liquidationValue = fullValue;
reserveCost = debt - liquidatorCost;
```

The liquidatorCost = 0 branch produces a strictly negative-EV action for any rational actor once gas > fullValue. The comment calls this "free liquidation", but with a positive gas cost it is effectively no liquidation. Combined with the absence of a write-off or keeper-incentive path — verified by enumerating every state-changing V4Vault entry point (create, notifyERC721Received, transferLoan, approveTransform, transform, borrow, decreaseLiquidityAndCollect, repay, liquidate, remove, deposit, mint, withdraw, redeem, setTokenConfig, setTransformer, setLimits, setReserveFactor, setReserveProtectionFactor, setHookAllowList, setEmergencyAdmin, setTokenDebtLimit, withdrawReserves) — the state persists indefinitely.

### Internal Pre-conditions

- A loan's fullValue drops to ≤ debt × MAX_LIQUIDATION_PENALTY_X32 (10% of debt).

- The position's dust value is below the gas cost of the liquidate transaction.

- debtSharesTotal includes the phantom debt; lendExchangeRateX96 is derived from it.

- No keeper with a subsidy budget or governance-funded gas reimbursement is actively calling liquidate on dust positions.

### External Pre-conditions

The position's underlying pool requires a sharp adverse move (or a sequence of partial liquidations) to reach the dust band. No adversarial actor is required; a normal market move is sufficient.

### Attack Path

- A vault-held position falls into the dust band (fullValue ≤ 10% × debt).

- Every rational liquidator observes liquidatorCost = 0 and skips (gas > reward).

- Interest continues to accrue: _calculateGlobalInterest uses unreduced debtSharesTotal, so lendExchangeRateX96 continues to grow.

- New lenders deposit at the inflated rate; their shares are immediately worth less than the reported lendExchangeRateX96 implies.

- Months later, the position is finally liquidated (a batch keeper, a charity case, or a governance intervention).

- _handleReserveLiquidation drops lendExchangeRateX96 in a single step to reflect the accumulated shortfall.

- Every lender present at that instant absorbs the loss; earlier withdrawers were over-paid.

- Lenders suffer an approximate loss equal to the accumulated phantom debt at the moment of realization; the loss is concentrated on whoever is holding shares at that instant.

- Quantitative example on a vault with $1,000,000 in lender deposits and $100,000 of dust debt held for one year at ~10% APY, with no reserves:

- Phantom interest accrued on the stuck loan (~$10,000) is erased on realization.

- The $100,000 unrecoverable principal is drawn against reserves (≈ $0 if the vault is fully utilized).

- Net effect: a lendExchangeRateX96 haircut approximating $100k / $1M = 10% of lender principal.

- There is no attacker profit — the loss is a griefing-shaped structural drift, not theft. New depositors overpay for shares; long-standing depositors take the eventual hit. The protocol's AUDIT-ACCEPTED-TOTAL-INSOLVENCY-REDEPLOY covers the terminal rate = 0 case; it does not cover this intermediate phantom-accrual regime.

### Fix

Add a small flat reward to liquidatorCost, funded from the reserve budget or from the position's own residual collateral, making dust liquidations gas-neutral to a rational actor. Bounds the persistence window but keeps the passive-liquidation model.

### POC

Add below code in `test/poc/PoC_F31_DustBadDebt.t.sol` and run `forge test --match-contract PoC_F31_DustBadDebt -vvv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.30;

import {console} from "forge-std/console.sol";
import {V4VaultLocalBase} from "test/vault/support/V4VaultLocalBase.sol";
import {InterestRateModel} from "src/vault/InterestRateModel.sol";
import {IVault} from "src/vault/interfaces/IVault.sol";

/// @title PoC — F-3.1: Dust bad debt persists, phantom debt accrues, and realization
///        is a single-step rate haircut for all remaining lenders
///
/// @notice EXPECTED VS ACTUAL
///   Expected: a dust-unhealthy position is liquidated for a positive keeper reward;
///   the lender's claim is backed by cash + recoverable collateral; a new depositor
///   has no exposure to the phantom loan.
///   Actual:   the dust branch pays 0 to a keeper → no liquidation → the phantom
///   debt inflates the persisted lender rate → a new depositor inherits it → the
///   eventual realization drops the rate in a single step, destroying ≥25% of
///   lender principal.
///
/// @notice ASSUMPTIONS
///   - Deployed mainnet InterestRateModel: base 0%, mult 5%, jump 109%, kink 80%.
///     (Mirrors `script/DeployMainnet.s.sol`.)
///   - `reserveFactor = 0` (the `V4VaultLocalBase` default; the deployed mainnet
///     config uses 10%). With reserves absent, the full shortfall lands on lenders —
///     this is the worst-case demonstration. With 10% reserves the haircut shrinks
///     but remains material; the mechanism is identical.
///   - 80% initial utilization (borrow 800 of 1000 supplied) so the rate is at the
///     deployed kink and the accrual effect is measurable within the test horizon.
///   - `30 days` idle: enough for the persisted rate to visibly inflate. The
///     material loss claim does not rest on the accrual magnitude; it rests on the
///     realization haircut, which is bounded only by the phantom principal.
///   - `fullValue = 1e15` on a `debt ≈ 800e18`: ratio 1.25e-6, far below the 10%
///     dust threshold. `fullValue` is nonzero on purpose — zero collateral is a
///     different (and broader) pathological case; this PoC isolates the dust band.
///   - `lender2` deposits 100e18 so the pro-rata share of phantom debt exceeds 20%
///     of their deposit; smaller deposits make the over-charge dust.
///
/// @notice REGRESSION MARKER
///   This test ASSERTS buggy behaviour and PASSES today. It FAILS once any of the
///   following fixes land. When fixed, update per the markers in the body:
///     - positive `liquidatorCost` in the dust branch (minimum keeper reward), OR
///     - `writeOffLoan` / `sweepBadDebt` / `forceLiquidate` on the vault, OR
///     - any mechanism that prevents phantom debt from persisting in
///       `debtSharesTotal` across accruals.
///
/// @dev Fork-free. `V4VaultLocalBase`: real PoolManager/PositionManager, mock oracle
///      (`setMockPositionValue`), mock asset. All helpers (`_deposit`, `_createLoan`,
///      `_borrow`) are inherited.
contract PoC_F31_DustBadDebt is V4VaultLocalBase {
    uint256 constant Q64 = 2 ** 64;

    function setUp() public override {
        super.setUp();
        InterestRateModel(address(vault.interestRateModel())).setValues(
            0,                // base   = 0% APY
            Q64 * 5 / 100,    // mult   = 5% APY
            Q64 * 109 / 100,  // jump   = 109% APY
            Q64 * 80 / 100    // kink   = 80%
        );
        // reserveFactor stays at the base default (0). Documented above.
    }

    function test_RegressionMarker_DustBadDebtPersistsAndAccrues() public {
        // ================================================================
        // Step 0: 80% utilization, healthy loan, then drop into the dust band
        // ================================================================
        _deposit(lender, 1000e18);
        oracle.setMockPositionValue(1000e18);
        uint256 tokenId = _createLoan(uint128(1000e18));
        _borrow(tokenId, 800e18);

        // fullValue ≈ 1.25e-6 × debt — well inside the 10% dust band.
        oracle.setMockPositionValue(1e15);

        (uint256 debt, uint256 fullValue,, uint256 liquidationCost, uint256 liquidationValue) =
            vault.loanInfo(tokenId);
        console.log("Step 0: debt          =", debt);
        console.log("Step 0: fullValue     =", fullValue);
        console.log("Step 0: liquidatorCost=", liquidationCost);
        console.log("Step 0: liquidatorVal =", liquidationValue);

        // ================================================================
        // Step 1: dust branch returns zero reward → no rational liquidator
        // ================================================================
        // REGRESSION MARKER: after a minimum-keeper-reward fix, this becomes
        // assertGt(liquidationCost, 0).
        assertEq(liquidationCost, 0, "BUG: dust branch pays zero keeper reward");
        assertEq(liquidationValue, fullValue, "BUG: dust collateral is all the keeper would receive");
        assertGt(debt, 100 * fullValue, "scenario: fullValue is below 10% of debt (dust)");

        // ================================================================
        // Step 2: phantom debt persists and accrues; lender rate is inflated
        // ================================================================
        uint256 debtBeforeWarp = debt;
        uint256 rateBefore = vault.lastLendExchangeRateX96();

        vm.warp(block.timestamp + 30 days);
        // Force accrual to persist. `loanInfo` and the getters below are views;
        // only a state-changing call runs `_updateGlobalInterest`. Without this
        // line, `lastLendExchangeRateX96` returns the pre-warp value and the
        // next assertion fails for the wrong reason.
        vault.setReserveFactor(vault.reserveFactorX32());

        (uint256 debtAfterWarp,,,,) = vault.loanInfo(tokenId);
        uint256 rateAfter = vault.lastLendExchangeRateX96();
        console.log("Step 2: debt after 30d=", debtAfterWarp);
        console.log("Step 2: rate after 30d=", rateAfter);

        // REGRESSION MARKER: after the fix (write-off or keeper reward that
        // prevents persistence) the persisted rate should not inflate on
        // unrecoverable debt.
        assertGt(debtAfterWarp, debtBeforeWarp, "BUG: phantom dust debt accrues interest");
        assertGt(rateAfter, rateBefore, "BUG: lender rate inflated by phantom interest");

        // ================================================================
        // Step 3: the inflated claim is unrealizable — a full redeem reverts
        // ================================================================
        (, uint256 lent, uint256 balance,,,) = vault.vaultInfo();
        console.log("Step 3: lender claim  =", lent);
        console.log("Step 3: cash + dust   =", balance + fullValue);
        console.log("Step 3: unrealizable  =", lent - (balance + fullValue));

        assertGt(lent, balance + fullValue, "BUG: lender claim exceeds realizable assets");
        assertGt(
            lent - (balance + fullValue),
            debtBeforeWarp / 2,
            "BUG: unrealizable gap is at least half the phantom principal"
        );

        uint256 lenderShares = vault.balanceOf(lender);
        assertGt(lenderShares, 0, "baseline: lender holds shares");
        vm.prank(lender);
        vm.expectRevert(abi.encodeWithSignature("InsufficientLiquidity()"));
        vault.redeem(lenderShares, lender, lender);

        // ================================================================
        // Step 4: a new depositor joins at the inflated rate — inherits the phantom
        // ================================================================
        // lender2 has no relationship to the phantom loan. They pay fair value at
        // deposit time (the inflated rate is applied to everyone) and inherit the
        // eventual shortfall pro-rata.
        address lender2 = makeAddr("lender2");
        uint256 depAmount = 100e18; // sized so lender2's pro-rata share of the
                                    // 800e18 phantom debt exceeds 20% of deposit
        _deposit(lender2, depAmount);

        uint256 lender2ClaimAtDeposit = vault.lendInfo(lender2);
        console.log("Step 4: lender2 shares     =", vault.balanceOf(lender2));
        console.log("Step 4: lender2 claim      =", lender2ClaimAtDeposit);

        // ================================================================
        // Step 5: realization — single-step rate haircut for all lenders
        // ================================================================
        address keeper = makeAddr("keeper"); // no funds needed: liquidatorCost == 0
        uint256 rateBeforeRealization = vault.lastLendExchangeRateX96();
        (, uint256 lentBeforeRealization,,,,) = vault.vaultInfo();

        vm.prank(keeper);
        vault.liquidate(IVault.LiquidateParams(tokenId, 0, 0, keeper, block.timestamp, ""));

        uint256 rateAfterRealization = vault.lastLendExchangeRateX96();
        (, uint256 lentAfterRealization,,,,) = vault.vaultInfo();
        console.log("Step 5: rate drop          =", rateBeforeRealization - rateAfterRealization);
        console.log("Step 5: lent before        =", lentBeforeRealization);
        console.log("Step 5: lent after         =", lentAfterRealization);

        // REGRESSION MARKER: with a fix that prevents phantom accrual, the rate
        // drop is bounded (reserves + minority shortfall only).
        assertLt(
            rateAfterRealization,
            rateBeforeRealization,
            "BUG: realization is a single-step rate drop"
        );
        assertLt(
            rateAfterRealization,
            rateBeforeRealization * 3 / 4,
            "BUG: single-step haircut exceeds 25% of lender principal"
        );

        // ================================================================
        // Step 6: lender2's loss = over-charge inherited from phantom debt
        // ================================================================
        uint256 lender2ClaimAfter = vault.lendInfo(lender2);
        uint256 lender2Loss = lender2ClaimAtDeposit > lender2ClaimAfter
            ? lender2ClaimAtDeposit - lender2ClaimAfter
            : 0;
        console.log("Step 6: lender2 after      =", lender2ClaimAfter);
        console.log("Step 6: lender2 loss       =", lender2Loss);

        // REGRESSION MARKER: after the fix, lender2's loss should reflect only
        // honest market risk, not inherited phantom debt. Threshold (>20% of
        // deposit) captures "material" without over-fitting to a specific share.
        assertGt(
            lender2Loss,
            depAmount / 5,
            "BUG: new depositor loses >20% of deposit to inherited phantom debt"
        );

        // ================================================================
        // Step 7: no write-off path exists (documented, not asserted)
        // ================================================================
        // grep -rn "writeOff\|sweepBadDebt\|forceLiquidate" src/vault/V4Vault.sol
        // returns no matches. The absence is the finding; nothing to assert at
        // runtime.
    }
}
```


