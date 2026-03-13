# AGENTS.md

## Technical Standards

- Solidity Version: 0.8.26+
- Use `PoolKey` and `PoolId` types correctly.
- When mining hook addresses, ensure bits for `AFTER_INITIALIZE` and `AFTER_SWAP` are set.
- All internal swaps must have a recursion guard to prevent re-entrancy.

## Architecture

- Strictly adhere to the v4-core singleton architecture.
- All token accounting must use `poolManager.settle()` and `poolManager.take()` — never raw ERC20 transfers directly to/from the pool.
- The `_settle` pattern is: `poolManager.sync(currency)` → `currency.transfer(address(poolManager), amount)` → `poolManager.settle()`.
- The `_take` pattern is: `poolManager.take(currency, recipient, amount)`.

## Hook Permissions

- `getHookPermissions()` must exactly match the permission bits encoded in the deployed hook address.
- Currently active permissions: `afterInitialize`, `afterSwap`.
- Hook address must be mined with flags: `Hooks.AFTER_INITIALIZE_FLAG | Hooks.AFTER_SWAP_FLAG`.
- Any new permission (e.g. `beforeSwap`, `beforeSwapReturnDelta`) requires re-mining the hook address and updating `getHookPermissions()`.

## Re-entrancy / Recursion Guard

- All internal swaps triggered by hook callbacks (e.g. `executeOrder` inside `_afterSwap`) **must** be protected by an EIP-1153 transient-storage recursion guard.
- Pattern to follow (from `TakeProfitsHook.sol`):

```solidity
uint256 private constant EXECUTING_ORDER_SLOT =
    uint256(keccak256("TakeProfitsHook.executingOrder")) - 1;

// In the hook callback:
bool executing;
uint256 _guardSlot = EXECUTING_ORDER_SLOT;
assembly ("memory-safe") { executing := tload(_guardSlot) }
if (executing) return (this.afterSwap.selector, 0);

// In swapAndSettleBalances, wrapping poolManager.swap():
assembly ("memory-safe") { tstore(_guardSlot, 1) }
BalanceDelta delta = poolManager.swap(key, params, "");
assembly ("memory-safe") { tstore(_guardSlot, 0) }
```

- Use `uint256(keccak256("ContractName.slotPurpose")) - 1` to derive collision-resistant transient slot keys.
- Never use `sender == address(this)` as a re-entrancy guard — it can be spoofed.

## EIP-1153 Transient Storage

- Use `tstore`/`tload` (EVM Cancun, enabled in `foundry.toml` via `evm_version = "cancun"`) for any cross-function state that is scoped to a single transaction.
- `tstore`/`tload` cost ~100 gas each and reset automatically at transaction end — no manual cleanup needed on happy path.
- On revert, the EVM rolls back `tstore` writes, so no cleanup is needed on the error path either.

## NoOp Pattern (`beforeSwap` returning `BeforeSwapDelta`)

- To implement gas-shielding or async execution, prefer returning a non-zero `BeforeSwapDelta` from `_beforeSwap` to bypass the AMM's internal math.
- Enable this by setting `beforeSwap: true` and `beforeSwapReturnDelta: true` in `getHookPermissions()` and re-mining the hook address.
- `BeforeSwapDelta` and `BeforeSwapDeltaLibrary` are already imported in `TakeProfitsHook.sol` for this purpose.

## ERC20 Safety

- Always use `SafeERC20.safeTransferFrom` / `safeTransfer` for user-facing token intake.
- Never use unchecked `IERC20.transferFrom` — non-standard tokens may not return a bool.

## Type Safety

- Before casting `uint256 → int256`, guard with `require(amount <= uint256(type(int256).max), "overflow")`.
- Suppress the forge-lint `unsafe-typecast` warning with `// forge-lint: disable-next-line(unsafe-typecast)` only after the guard is in place.

## Testing

- Deploy hooks with `deployCodeTo("HookName.sol", abi.encode(...), hookAddress)` where `hookAddress = address(uint160(flags))`.
- `flags` must be `Hooks.AFTER_INITIALIZE_FLAG | Hooks.AFTER_SWAP_FLAG` (and any additional flags matching `getHookPermissions()`).
- Run `forge test -vv` to execute the full suite; all tests must pass with zero compiler errors.
- `forge build` must complete with zero errors and zero warnings (notes are acceptable).
