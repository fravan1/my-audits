# PrePO draft findings

---

## Finding 1

`DepositTradeHelper.sol`

_Issue_

- account has infinite approval to spend baseTokens & collateralTokens of user

- these permissions could be misused by a malicious contract that can call a low level function `transferFrom` and drain baseTokens and preCT collateral tokens of user

_Proof of Concept_

- Assuming a malicious contract knows the public address of an account that gave infinite approval to `DepositTradeHelper.sol`

- by passing in malicious low level call, we can capture the funds. Same logic for collateral token

```

    function exploit(address depositHelper, address token, address depositor) external {

        IERC20 tokenContract = IERC20(token);

        uint256 balance = tokenContract.balanceOf(depositor);

        if(balance > 0){
           (bool success, ) = depositHelper.call(abi.encodeWithSignature("transferFrom(address,address,uint256)", depositor, address(this), balance  ));
           require(success, "transfer failed");
        }
    }
```

_Recommendation_

Right after the swapRouter completes the swap - make the approvals for the contract 0

```
    ISwapRouter.ExactInputSingleParams memory exactInputSingleParams = ISwapRouter.ExactInputSingleParams(address(_collateral), tradeParams.tokenOut, POOL_FEE_TIER, msg.sender, tradeParams.deadline, _collateralAmountMinted, tradeParams.amountOutMinimum, tradeParams.sqrtPriceLimitX96);

    _swapRouter.exactInputSingle(exactInputSingleParams); // amountOut output is not used

    // At this stage, make approvals go to zero

    if (baseTokenPermit.deadline != 0) {
      IERC20Permit(address(_baseToken)).permit(msg.sender, address(this), 0, baseTokenPermit.deadline, baseTokenPermit.v, baseTokenPermit.r, baseTokenPermit.s);
    }

    if (collateralPermit.deadline != 0) {
      _collateral.permit(msg.sender, address(this), 0, collateralPermit.deadline, collateralPermit.v, collateralPermit.r, collateralPermit.s);
    }
```

---

## Finding 2

`DepositTradeHelper.sol`

_Issue_

Function seems to be incomplete - the tokenOutput from \_SwapRouter is currently owneed by DepositTradeHelper -> once we receive tokenOutput, the tokens need to be sent back to msg.sender

`exactInputSingle` function used returns the `amountOut` of output token - this return value is currently not being used by the function

```
ISwapRouter.ExactInputSingleParams memory exactInputSingleParams = ISwapRouter.ExactInputSingleParams(address(_collateral), tradeParams.tokenOut, POOL_FEE_TIER, msg.sender, tradeParams.deadline, _collateralAmountMinted, tradeParams.amountOutMinimum, tradeParams.sqrtPriceLimitX96);

    _swapRouter.exactInputSingle(exactInputSingleParams);

```

Also, token output is currently in the `DepositTradeHelper` contract -> this needs to be transferred back to `msg.sender` - that part of code is missing.

_Recommendation_

Helper should complete the loop -> by transfering tokens returned by Uniswap V3 router to msg.sender as follows

```

    uint256 tokenOutput = _swapRouter.exactInputSingle(exactInputSingleParams); // amountOut output is not used

    if (tokenOutput > 0) {
      bool success = IERC20(tradeParams.tokenOut).transfer(msg.sender, tokenOutput);
      require(success, "swapped token transfer failed");
    }

```

---

## Finding 3 - DONE

Possible Denial of Service when users try to withdraw collateral

[In the `managerWithdraw` function](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/Collateral.sol#L80), user with `MANAGER_WITHDRAW_ROLE` can withdraw funds from Collateral vault subject to a [minimum reserve requirement condition](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/ManagerWithdrawHook.sol#L17)

At this stage, we are not checking if the balance left after withdrawal is sufficient to honor any collateral withdrawals by users.

In the extreme case where minimum Reserve < collateral withdrawal of a user in a single period, users will face a denial of service. Ofcourse the edge case here is dependent on `userWithdrawalCapPerPeriod` and `minimumReserve` but such an edge case is conceptually possible.

_Proof of Concept_

Suppose vault had 10 depositors of 100 USDC each. `userWithdrawLimitPerPeriod` is 80 USDC and `minimumReserve` is 5%. A user with `MANAGER_WITHDRAW_ROLE` withdraws 95% of base tokens from vault. Bob, a depositor, tries to withdraw maximum of 80 USDC, gets rejected because of insufficient balance in the vault

_Recommendation_

Just like `globalNetDepositAmount` being tracked in `DepositRecord` contract, a `globalUserDepositAmount` should be tracked that is a cumulative sum of all user deposits (excluding deposits made by protocol itself).

Check following condition before allowing manager to withdraw funds

`minimumReserve * globalNetDepositAmount >= globalUserDepositAmount`

This will ensure that vault will always have enough balance to meet any withdrawal requests coming from users

---

## Finding 4 (DONE)

Checks for Global and User Withdraw Limit Per Period are missing for the first withdrawal request right AFTER period length expires and a new period begins

In the [WithdrawHook contract](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/WithdrawHook.sol#L59), when `block.timestamp > lastGlobalPeriodReset + globalPeriodLength`, two variables `lastGlobalPeriodReset` and `globalAmountWithdrawnThisPeriod` are being reset.

However, for the first withdrawal request right when the two above variables are being reset, we are not checking if `_amountBeforeFee` exceeds `globalWithdrawLimitPerPeriod`

Same problem also exists for `lastUserPeriodReset` and ` userToAmountWithdrawnThisPeriod[_sender]` variables

If a user makes a large withdrawal request right after a period expires, that request will currently be honored. This is in direct violation of the designed logic that puts restriction on global withdrawal and user specific withdrawal in each period

_Proof of Concept_

    Suppose global withdrawal limit per period if 100 ETH and each period is for 100 seconds. Let's say current timestamp if 10000 -> when the timestamp hits 10100, `globalAmountWithdrawnThisPeriod` is being reset to `_amountBeforeFee`. Say Bob places a withdrawal request of 101 ETH -> this request gets processed even though it violates the per period withdrawal limit

Same problem exists for user wise limits.

---

## Finding 5 (DONE)

Access control for `hook` function in `RedeemHook` Contract is inconsistent with the implementation. Since the function involves a transfer of fees to Treasury, I've marked it as MEDIUM RISK

`RedeemHook` checks if sender is in a list of pre-approved accounts in `accountList`. Access control modifier for the `hook` function (`onlyAllowedMsgSenders`) allows any allowed msg sender to access this function. However, the function wraps a `IPrePOMarket` inteface on `msg.sender` - this ineffect means that the hook is expecting `msg.sender` to be `PrePOMarket` address

_Proof of Concept_

Bob is in the list of accounts of `_allowedMsgSenders` and hence `onlyAllowedMsgSenders` allows Bob to access the `redeemHook`.

However, Bob's account is not a `PrePOMarket` - it will not return collateral address when `getCollateral` function is called in Line 21 & hence `transferFrom` function can never work when Bob calls this function.

_Recommendation_

Since `redeemHook` will only be called from within the `redeem` function of `PrePOMarket`, appropriate access control is to only allow `PrePOMarket` address to access the `redeemHook`

## Finding 6 (DONE)

_Issue_
Possible Denial of Service to an existing LP if protocol owner accidentally resets the `AccountList` and uploads a different list of `accounts` that don't include that LP.

Once accepted as LP who minted Long/Short tokens in excchange for preCT, said LP must not lose access to `PrePOMarket` redeem functionality

_Proof of Concept_
Bob's account was initially in the `accountList` when `resetIndex` in [`AccountList` contract] (https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/AccountList.sol#L8) is set to 0.

Bob subsquently deposited USDC collateral and minted Long/Short tokens for a given prePO market. Thereafter protocol owner resets the `accountList` and misses to include `Bobs` address in the new list -> the `resetIndex` now shifts to 1 and `isIncluded()` function returns `false` for Bobs address

When Bob tries to redeem his tokens, `redeemHook` reverts with an error in [RedeemHook.sol](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/RedeemHook.sol#L18) in this line

```
    require(_accountList.isIncluded(sender), "redeemer not allowed");
```

As a result, Bob can never redeem his tokens. This denial of access to an existing LP by actions of owner increases centralization

_Recommendation_

`DepositRecord` address can be passed into `redeemHook`. This contains a state of all existing users (`userToDeposits`) -> if `userToDeposits[sender]>0`, user is currently an active LP with posted collateral -> in that case, existing LP should take precedence over `accountList` entries

Following condition can be updated

from

```
    require(_accountList.isIncluded(sender), "redeemer not allowed");

```

to

```
 require(_accountList.isIncluded(sender) || depositRecord.userToDeposits[sender] > 0, "redeemer not allowed");
```

## Finding 7 (DONE)

_Issue_

Inconsistent implementation of `expiryTime` in PrePOMarket

In line 199 of [IPrePOMarket](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol#L199), documentation says that `expiryTime` is not enforced and instead, market expiry occurs once `finalLongPayout` is set.

However, in the [`setFinalLongPayout` implementation](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/PrePOMarket.sol#L119) on Line 119 in `PrePOMarket.sol`, when finalPayout is being set, there is no update of `expiryTime` - `expiryTime` is only initialized in constructor and never changes again.

Based on comments above, `getExpiryTime` should reflect the timestamp when the final payout was set but implementation shows otherwise

_Recommendation_

Documentation is confusing - if expiry time should not be changed, inline comments should reflect that. Conversely, if inline comments are correct, then `expiryTime` should be reset when `setFinalPayout` function is called.

## Finding 8 (DONE)

_Issue_

Code Documentation errors

Inline documentation of code has following minor errors that can be corrected

1. Natspec for `IPrePOMarket` event has [duplicate entries](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol#L20)

2. Parameter definition for `amountAfterFee` in [`Redemption` event](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol#L39) is incorrect

3. Comments in [`IMarketHook`](https://github.com/prepo-io/prepo-monorepo/blob/3541bc704ab185a969f300e96e2f744a572a3640/apps/smart-contracts/core/contracts/interfaces/IMarketHook.sol#L16) incorrectly label parameters

_Recommendations_

1. Remove a duplicate line in Natspec ` * @param shortToken Market Short token address` in line 20 of `IPrePOMarket`
2. Replace ` * @param amountAfterFee The amount of Long/Short tokens minted` with ` * @param amountAfterFee The amount of Long/Short tokens redeemed` in line 39 of `IPrePOMarket`
3. Replace `amountBeforeFee` with `amountAfterFee` in line 16 of `IMarketHook`
