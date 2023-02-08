### H01 - Issue Header

**Vulnerability**

- There are some ERC20 tokens that deduct a fee on every transfer call. If these tokens used as baseToken then
  - When depositing into the Collateral contract, the recipient will receive collateral token more than what they should receive.
  - The DepositRecord contract will track wrong user deposit amounts and wrong globalNetDepositAmount as the added amount to both will be always more than what was actually deposited.
  - When withdrawing from the Collateral contract, the user will receive less baseToken amount than what they should receive.
  - The treasury will receive less fee and the user will receive more PPO tokens that occur in DepositHook and WithdrawHook.

**Code**
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L45-L61
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L64-L78

**Proof of Concept**

**Mitigation**

- Consider calculating the actual amount by recording the balance before and after.

```
uint256 balanceBefore = baseToken.balanceOf(address(this));
baseToken.transferFrom(msg.sender, address(this), _amount);
uint256 balanceAfter = baseToken.balanceOf(address(this));
uint256 actualAmount = balanceAfter - balanceBefore;
```

- Use only the difference in balance for calculating amount of collateral tokens minted

**Key learning**

This is an important point - if code assumes that amount of deposit == amount of collateral tokens, there is always a possibility that tokens that automatically deduct fees can break the accounting logic. This is a practice I need to put in place - always check if difference in balances is used OR direct amounts are used. Its always more robust to use balance differences
