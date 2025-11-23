# RebateFi Hook - Audit Report

Date: 2025-11-23

Repository: `2025-11-rebatefi-hook`

Summary: This audit inspects `src/RebateFiHook.sol` (the ReFi swap rebate hook) and supporting files. I performed a code review and made minimal, targeted fixes. I could not run the test suite locally because `forge` was not available in the environment.

Findings
1) Incorrect pool membership check in `_beforeInitialize`
   - Description: The contract originally checked `currency1` twice instead of verifying that either `currency0` or `currency1` equals the ReFi token. This could cause valid pools containing ReFi to be rejected (or invalid pools accepted depending on usage).
   - Risk: High — hooks may fail to initialize for valid pools, preventing intended usage.
   - Reproduction: Try to initialize a pool where ReFi is `currency0` and observe `ReFiNotInPool` revert.
   - Fix: Check both `currency0` and `currency1` for equality with the ReFi token. (Fixed)

2) Swap-direction logic inverted in `_isReFiBuy`
   - Description: The logic that determines whether a swap is a ReFi buy or sell was inverted. The hook would treat sells as buys and buys as sells.
   - Risk: Critical — fees would be applied to the wrong direction (rewards vs penalization inverted), completely breaking the intended economic model.
   - Reproduction: Create a pool with ReFi as `currency1` and perform a `zeroForOne` swap that results in receiving ReFi; the hook would incorrectly classify it.
   - Fix: Flip the boolean logic so that when `ReFi` is `currency0` the buy condition is `!zeroForOne`, otherwise `zeroForOne`. (Fixed)

3) `TokensWithdrawn` event argument order
   - Description: The `withdrawTokens` function emitted `TokensWithdrawn(to, token, amount)` while the event is declared as `(token, to, amount)`.
   - Risk: Low — only an event logging mismatch, but can confuse off-chain monitoring/analytics.
   - Fix: Emit the event with correct argument order. (Fixed)

4) Fee reporting / units ambiguity (observational)
   - Description: The contract logs a `feeAmount` computed as `(swapAmount * sellFee) / 100000` for the event only. The denominator / unit conventions are not explicitly documented in the contract and depend on `LPFeeLibrary` semantics.
   - Risk: Medium — mismatch between human-readable reports and actual fee semantics can cause confusion for integrators and tests.
   - Recommendation: Document fee units clearly (e.g., basis points vs parts-per-1e6) and, if necessary, compute human-readable amounts using the same denominator the pool expects.

5) `Ownable(msg.sender)` usage (note)
   - Description: The constructors in both `ReFi.sol` and `RebateFiHook.sol` call `Ownable(msg.sender)` when inheriting. OpenZeppelin's `Ownable` typically does not accept an argument. This may be intentional depending on the OZ version used in the environment, but it's worth validating.
   - Risk: Medium/Build — if the installed OpenZeppelin version does not expose a constructor with an address argument, compilation will fail.
   - Recommendation: Ensure the project's OpenZeppelin version supports this usage or change to the canonical pattern.

Other observations & recommendations
- Add an event for fee changes in `ChangeFee` to aid monitoring.
- Consider adding explicit safe checks in `withdrawTokens` for `token == address(0)` (ETH) and supporting ETH withdrawals separately.
- Add unit tests that assert fees are charged in the correct direction (both buy and sell), and tests that intentionally create pools where ReFi is `currency0` and where it is `currency1`.

Actions taken
- Fixed `_beforeInitialize` to check both currencies.
- Fixed `_isReFiBuy` logic to correctly detect buy vs sell.
- Fixed `TokensWithdrawn` event parameter ordering.

Files modified
- `src/RebateFiHook.sol` — applied the above fixes.

Next steps for you (recommended)
1. Install Foundry (`foundryup`) and run `forge test -vv` in the repo root:


2. Run static analysis (slither/solhint) and a linter to ensure no other issues.
3. Add tests specifically covering:
   - Pools where ReFi is `currency0` and `currency1` (confirm `beforeInitialize` allows both)
   - Confirm `buyFee` applies for buys and `sellFee` for sells (assert event emissions and pool fee values)
   - Edge cases for `withdrawTokens` and fee boundary values

4. If everything passes, prepare a PR with the fixes and include the audit report.

