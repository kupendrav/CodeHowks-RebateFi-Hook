# RebateFi Hook – Submission Pack

This file contains ready-to-paste vulnerability writeups for the CodeHawks First Flight contest. Each entry already follows the required "Root + Impact" template.

---

## 1. ReFi sells misclassified as buys, letting dumpers dodge the premium fee (High)

### Root + Impact

#### Description
* Normal behavior: When users buy the designated ReFi token, the hook should apply `buyFee` (typically 0%) so buys are subsidized; when users sell ReFi, it should apply `sellFee` (e.g., 0.3%) to discourage dumping.
* Actual behavior: `_isReFiBuy` misclassifies every swap whenever ReFi is `currency0`, so sells receive the zero-fee path and buys get penalized. This inverts the entire incentive model and lets dumpers avoid the premium fee.

```solidity
// src/RebateFiHook.sol
function _isReFiBuy(PoolKey calldata key, bool zeroForOne) internal view returns (bool) {
    bool IsReFiCurrency0 = Currency.unwrap(key.currency0) == ReFi;
    @> if (IsReFiCurrency0) {
    @>     return zeroForOne;        // BUG: direction is inverted
    @> } else {
    @>     return !zeroForOne;
    @> }
}
```

#### Risk
**Likelihood**
* Any pool where ReFi is `currency0` (roughly half of deployments) hits this bug immediately.
* Users only need to perform normal Uniswap swaps; no special permissions or conditions are required.

**Impact**
* Sellers never pay the configured premium fee, eliminating protocol revenue and removing the anti-dump mechanism.
* Buyers are overcharged, discouraging accumulation and undermining the hook’s stated economic goal.

### Proof of Concept
Deploy a pool with `currency0 == ReFi`, perform a standard sell swap (`zeroForOne == true`), and log the fee value returned by the hook—you’ll observe it equals `buyFee` instead of `sellFee`.

```solidity
function test_Sell_BypassesFee_WhenReFiIsCurrency0() external {
    vm.deal(user1, 1 ether);
    vm.startPrank(user1);
    reFiToken.approve(address(swapRouter), type(uint256).max);

    SwapParams memory sell = SwapParams({
        zeroForOne: true,              // selling ReFi because it is currency0
        amountSpecified: -int256(0.01 ether),
        sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
    });
    uint24 fee = swapRouter.swap(key, sell, DEFAULT_SETTINGS, ZERO_BYTES);

    assertEq(fee, rebateHook.buyFee(), "sell wrongly charged buyFee");
}
```

### Recommended Mitigation
Flip the logic so “buy” means the user receives ReFi; when ReFi is `currency0`, that happens only when `zeroForOne == false`.

```diff
 function _isReFiBuy(PoolKey calldata key, bool zeroForOne) internal view returns (bool) {
     bool IsReFiCurrency0 = Currency.unwrap(key.currency0) == ReFi;
-    if (IsReFiCurrency0) {
-        return zeroForOne;
-    } else {
-        return !zeroForOne;
-    }
+    if (IsReFiCurrency0) {
+        return !zeroForOne;   // zeroForOne means selling currency0 (ReFi)
+    } else {
+        return zeroForOne;    // zeroForOne means receiving currency1 (ReFi)
+    }
 }
```

Severity suggestion: **High**.

---

## 2. Valid pools rejected because beforeInitialize checks currency1 twice (Medium)

### Root + Impact

#### Description
* Normal behavior: `beforeInitialize` should allow any pool that contains the designated ReFi token—whether ReFi is listed as `currency0` or `currency1`—so LPs can pair ReFi against any other asset in either order.
* Actual behavior: The pool-membership gate only checks `currency1` twice (copy/paste mistake). Any pool where ReFi sits in the `currency0` slot reverts with `ReFiNotInPool`, even though the ReFi token is present.

```solidity
// src/RebateFiHook.sol
function _beforeInitialize(address, PoolKey calldata key, uint160) internal view override returns (bytes4) {
    if (
    @>      Currency.unwrap(key.currency1) != ReFi &&
    @>      Currency.unwrap(key.currency1) != ReFi
    ) {
        revert ReFiNotInPool();
    }
    return BaseHook.beforeInitialize.selector;
}
```

#### Risk
**Likelihood**
* Pool deployers choose token ordering arbitrarily; whenever ReFi happens to be passed as `currency0`, initialization hits this revert.
* Nothing in the UI or scripts enforces ReFi=token1, so this happens frequently in practice.

**Impact**
* Legitimate pools cannot be initialized, preventing liquidity provisioning for those pairs.
* Contest goals (incentivizing buys, penalizing sells) can’t be met for any blocked pool, effectively halting adoption.

### Proof of Concept
Swap the order of `currency0`/`currency1` when calling `initPool`; the hook rejects it even though ReFi is present.

```solidity
function test_InitRevertsWhenReFiIsCurrency0() external {
    (PoolKey memory key,) = initPool(
        reFiCurrency,        // currency0
        ethCurrency,         // currency1
        rebateHook,
        LPFeeLibrary.DYNAMIC_FEE_FLAG,
        SQRT_PRICE_1_1_s
    );

    vm.expectRevert(ReFiSwapRebateHook.ReFiNotInPool.selector);
    manager.initialize(key, SQRT_PRICE_1_1_s, ZERO_BYTES);
}
```

### Recommended Mitigation
Check both sides of the pair (token0 and token1) instead of checking the same slot twice.

```diff
 function _beforeInitialize(address, PoolKey calldata key, uint160) internal view override returns (bytes4) {
-    if (Currency.unwrap(key.currency1) != ReFi &&
-        Currency.unwrap(key.currency1) != ReFi) {
+    if (Currency.unwrap(key.currency0) != ReFi &&
+        Currency.unwrap(key.currency1) != ReFi) {
         revert ReFiNotInPool();
     }
     return BaseHook.beforeInitialize.selector;
 }
```

Severity suggestion: **Medium**.

---

