# H01 - Protection buyers can front-run lending pool state updates to buy protection on lending pools that have just transitioned from `Active` to `LateWithinGracePeriod` state

## Summary

Protocol docs state that a `Cron Job` will run daily to execute `assetStates` on `DefaultStateManager`. `assesStates` is responsible for updating state of lending pool, active, late or default.

A protection buyer who knows the next payment date of lending pool can front-run this cron job & buy protection right after a lending pool transitions from `active` to `LateWithinGracePeriod` status. Protection sellers will end up taking exposure to high risk loans

## Vulnerability Detail

To buy protection, users call `buyProtection` that inturn calls `verifyProtection` on Pool Helper. This function does 2 checks

1. Checks if protection duration is greater than minimum duration & lesser than next period end cycle
2. Checks if underlying lending pool is active - [`_verifyLendingPoolIsActive`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/libraries/ProtectionPoolHelper.sol#L422) reverts if lending pool is either late or late but within grace period or defaulted

Protection buyer can access the [`isLendingPoolLateWithinGracePeriod`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/adapters/GoldfinchAdapter.sol#L94) in the `GoldfinchAdapter` contract to check if lending pool, for the first time, transitions from `Active` to `LateWithinGracePeriod`

User can easily calculate the next payment date by combining `getPaymentPeriodInDays` and `getLatestPaymentTimestamp` functions in `GoldfinchAdapter`. On this day, protection buyer just has to front run the `assessState` execution & buy protection for his entire position in Lending Pool if the pool goes into `late` state.

Since this transaction gets executed before state update, `_verifyLendingPoolIsActive` function allows protection purchase without reverting. Protocol sellers end up picking up huge risk without any additional reward.

## Impact

All transactions will be one-sided - there will be a rush of protection buyers covering their risky loans by exploiting the new information.

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/libraries/ProtectionPoolHelper.sol#L422

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/adapters/GoldfinchAdapter.sol#L94

## Tool used

Manual Review

## Recommendation

Every `buyProtection` call should mandatorily `assessStateBatch` function by sending the specific lending pool (`_pools` array will have length 1 to save gas) for which protection is being bought. This call should be before `__verifyLendingPoolIsActive` call to ensure latest pool status

# H02 - Malicious protection buyer can manipulate pool leverage ratio to block genuine protection buyers

## Summary

Protocol docs state that `naked protection buying` is not allowed - scenario where a protection buyer does not hold underlying loan position & yet makes a protection claim. This is currently enforced by verifying if protection buyer holds the NFT token when making a claim for protection payout.

Unfortunately, similar verification does not exist at the time of buying protection. This opens an attack vector by a malicious protection buyer who can keep transferring same NFT token across multiple addresses & keep buying protection in a calibrated manner to manipulate leverage ratio of protection pool.

Such an attack, although expensive for the attacker, can block genuine protection buyers. Newly launched pools are especially vulnerable to such attacks (refer to POC)

## Vulnerability Detail

When users buys protection, [`_verifyAndCreationProtection`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L795) verifies and creates a new protection. Note that there is no check to verify if the same NFT token is already used by a user who has active protection.

However, a check does exist to verify if the leverage rate, after adding protection, falls below the floor level. If it does, no new protection buyer can come in until further sellers capitalize the pool to increase leverage ratio beyond this floor.

```solidity
  function _verifyAndCreateProtection(
    uint256 _protectionStartTimestamp,
    ProtectionPurchaseParams calldata _protectionPurchaseParams,
    uint256 _maxPremiumAmount,
    bool _isRenewal
  ) internal {

    ....

        /// Step 1: Calculate & check the leverage ratio
    /// Ensure that leverage ratio floor is never breached
    totalProtection += _protectionPurchaseParams.protectionAmount;
    uint256 _leverageRatio = calculateLeverageRatio();
    if (_leverageRatio < poolInfo.params.leverageRatioFloor) {
      revert ProtectionPoolLeverageRatioTooLow(_leverageRatio);
    }
    ....
}

```

A new pool that has capital just above minimum capital, ie a pool that has just been upgraded to `Open` can be easily attacked by a user. Since the pool has just above minimum capital, capital needed to execute this attack will also be minimal.

**POC**

- A new pool with minimum capital of 100k USDC is launched (say Leverage Ratio Floor = 10%)
- Protection sellers capitalize the pool to 100k USDC
- Pool is now open to buyers
- Alice holds a NFT for loan position (say, 10k USDC) in Goldfinch lending pool that is part of current protection pool
- Alice waits for pool to be `Open`, ie leverage ratio falls below ceiling
- Alice creates a exploit contract that transfers NFT to a new address & buys protection in a loop until leverage ratio falls below floor value
- Every time a protection seller comes in with `X` capital, Alice can run above loop to buy `10X` protection (to keep leverage ratio under 10%)
- Bob, a genuine protection buyer, gets DOS'd as pool leverage falls below floor if his protection is added

## Impact

New pool is effectively dysfunctional as supply is centralized and manipulated by a single buyer. A well crafted attack can block new users from becoming protection buyers.

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L795

## Tool used

Manual

## Recommendation

A mapping of lendingPool -> nftTokenId -> buyer needs to be created and tracked in `ProtectionPool`. Combination of lendingPool & nftTokenID should have a unique buyer for active protections.

Every time a new buyer comes in, this mapping should be updated. Every time a protection expires, this mapping is reset to 0 address.

# H03 - Malicious user can cause a DOS attack on critical pool functions such as `accruePremiumAndExpireProtections`

## Summary

A malicious user can buy very small amount of protection against a given loan position. This can create a practically unlimited number of active protections.

Mission critical operations such as `accruePremiumAndExpireProtections` that are run via a `cron job` can be DOSed as the function loops over all active Protections for each lending pool.

## Vulnerability Detail

Note that for every protection, a new item is pushed to `protectionInfos` array. A malicious user can create an unbounded array by buying a small amount of protection each time.

```solidity

  function _verifyAndCreateProtection(
    uint256 _protectionStartTimestamp,
    ProtectionPurchaseParams calldata _protectionPurchaseParams,
    uint256 _maxPremiumAmount,
    bool _isRenewal
  ) internal {
    ...
    /// Step 5: Add protection to the pool
    protectionInfos.push(
      ProtectionInfo({
        buyer: msg.sender,
        protectionPremium: _premiumAmount,
        startTimestamp: _protectionStartTimestamp,
        K: _k,
        lambda: _lambda,
        expired: false,
        purchaseParams: _protectionPurchaseParams
      })
    );
    ...
}
```

`_accruePremiumAndExpireProtections` that is responsible to accrue premium to sellers and retire expired positions can run into large loops and revert with `out of gas` exception.

```solidity
  function _accruePremiumAndExpireProtections(
    LendingPoolDetail storage lendingPoolDetail,
    uint256 _lastPremiumAccrualTimestamp,
    uint256 _latestPaymentTimestamp
  )
    internal
    returns (
      uint256 _accruedPremiumForLendingPool,
      uint256 _totalProtectionRemoved
    )
  {
    ...
  uint256 _length = _protectionIndexes.length;
    for (uint256 j; j < _length; ) {
      uint256 _protectionIndex = _protectionIndexes[j];
      ProtectionInfo storage protectionInfo = protectionInfos[_protectionIndex];
        ...
          unchecked {
        ++j;
        }
    }
    ...
}
```

## Impact

Protection sellers will not be earn yield leading to a dysfunctional pool.

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L872

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L963

## Recommendation

Recommend having a `miniumProtectionPercent` variable as part of protection pool parameters. This should be a percentage of outstanding principal of a buyer. If set at say 10%, a 1000 USDC lender has to take a minimum protection of 100 USDC in a single transaction while a 1 million USDC lender has to take a minimum protection of 100k USDC

Alternate, and less preferable, is to cap the number of protection buy transactions per user

# H04 - A buyer can buy more protection than his oustanding principal on lending pool

## Summary

A user can keep buying protection multiple times for smaller amounts that cumulatively add up to more than his outstanding principal. Protocol doesn't track cumulative protection purchased by a buyer when buying new protection.

This can lead to loss of premium for naive/first-time protection buyers who might (wrongly) think that they get more payout if they buy more protection. Accidentaly buying (users not realizing/forgeting that they have already purchased) also ends up in buyers paying additional premium for no additional benefit.

## Vulnerability Detail

When users buy protection `verifyProtection` gets called that checks protection duration, active lending pool and lastly, protection amount. `canBuyProtection` function in `ReferenceLendingPools` checks if protection amount in `purchaseParams.protectionAmount` is less than outstanding principal of buyer.

Here is snapost of `canBuyProtection` function in `ReferenceLendingPool`

```solidity

 function canBuyProtection(
    address _buyer,
    ProtectionPurchaseParams calldata _purchaseParams,
    bool _isRenewal
  )
    external
    view
    override
    whenLendingPoolSupported(_purchaseParams.lendingPoolAddress)
    returns (bool)
  {
    ...
        // -audit comparing protectionAmount sent in purchase params with remaining principal
         return
      _purchaseParams.protectionAmount <=
      calculateRemainingPrincipal(
        _purchaseParams.lendingPoolAddress,
        _buyer,
        _purchaseParams.nftLpTokenId
      );

    ...

}

```

Note above that the only comparison is if the current protection sought by buyer is less than outstanding principal. Buyer might have already bought protection upto the principal previously - since that amount is not tracked here, `canBuyProtection` returns true.

## Impact

Maximum payout offered by protocol is equal to the outstanding principal of buyer in the underlying lending pool. Buying more protection is just a complete loss of premium to protection buyers with no benefit whatsoever.

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/libraries/ProtectionPoolHelper.sol#L70

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ReferenceLendingPools.sol#L162

## Recommendation

Cumulative protection amount for each user needs to be tracked inside `ProtectionBuyerAccount` struct. Recommend having following mapping similar to `lendingPoolToPremium`

```solidity
mapping(address=> uint256)lendingPoolToProtection;

```

`lendingPoolToProtection` should track the total active protection for a given buyer against a specific lending pool address. Condition inside `canBuyProtection` should check the cumulative protection (instead of current protection) exceeds remaining principal

```solidity

      return
      _purchaseParams.protectionAmount + lendingPoolPremium[_buyer] <=
      calculateRemainingPrincipal(
        _purchaseParams.lendingPoolAddress,
        _buyer,
        _purchaseParams.nftLpTokenId
      );

```

# H05 - Protocol locks up capital higher than outstanding principal of buyer

## Summary

I've made an earlier submission stating that a user can buy protection for an amount greater than principal outstanding in lending pool. This is possible by buying protection on smaller amounts multiple times that cumulatively adds up to an amount greater than outstanding principal. Current issue is an extension of the same issue - but this time, vulnerability exists in logic for locking capital.

A similar possibility exists in the `lockCapital` function where a protocol can lock capital greater than the current outstanding. Just like the protection amount should be capped at principal outstanding, locked capital (amount taken from protection sellers capital) should also be capped at principal outstanding.

No such checks exist in `lockCapital` function resulting in losses for protection sellers who lose access to capital much higher than actual loss faced by protection buyers

## Vulnerability Detail

`lockCapital` function loops over each active protection index - for every active protection, minimum of protection amount or remaining principal (calculated by `GoldFinchAdapter`) is added to total lockedAmount. This amount is then reduced from `totalSTokenUnderlying` and will not be accessible to protection sellers.

Code snippet below shows the calculations:

```solidity
  function lockCapital(address _lendingPoolAddress)
    external
    payable
    override
    onlyDefaultStateManager
    whenNotPaused
    returns (uint256 _lockedAmount, uint256 _snapshotId)
  {

    ...

     uint256 _length = activeProtectionIndexes.length();
    for (uint256 i; i < _length; ) {
      /// Get protection info from the storage
      uint256 _protectionIndex = activeProtectionIndexes.at(i);
      ProtectionInfo storage protectionInfo = protectionInfos[_protectionIndex];

      /// Calculate remaining principal amount for a loan protection in the lending pool
      uint256 _remainingPrincipal = poolInfo
        .referenceLendingPools
        .calculateRemainingPrincipal(
          _lendingPoolAddress,
          protectionInfo.buyer,
          protectionInfo.purchaseParams.nftLpTokenId
        );

      /// Locked amount is minimum of protection amount and remaining principal
      uint256 _protectionAmount = protectionInfo
        .purchaseParams
        .protectionAmount;
      uint256 _lockedAmountPerProtection = _protectionAmount <
        _remainingPrincipal
        ? _protectionAmount
        : _remainingPrincipal;

      _lockedAmount += _lockedAmountPerProtection; //- audit if cumulative protection > remaining principal, _lockedAmount will be higher than actual


      unchecked {
        ++i;
      }

    }

   ...

  }
```

If smaller protection amounts are purchased multiple times that cumulatively add up to be greater than principal outstanding, for every active protection, protection amount would be added to `_lockedAmount`. There is no check to see if the cumulative `_lockedAmount` for each buyer overflows beyond the `remainingPrincipal` for that buyer.

## Impact

Consider the POC

- Alice has a loan position of 100k USDC in GoldFinch Lending Pool
- Alice has bought protection of 50k USDC 3 times totaling to 150k USDC (Nothing stops her from doing this as cumulative protection amount is currently not tracked for verification)
- Assume that specific Lending Pool goes from `Active` status to `Late`
- Also assume that remaining principal for Alice is still 100k
- When `LockedCapital` is called on all 3 active protections, a sum of $150k gets locked in protocol. When maximum payout by protocol should have been only 100k

Protection sellers cannot withdraw the additional $50k USDC locked in the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L401

## Recommendation

`lockCapital` function should calculate `_lockedAmount` for each buyer. Cumulative sum of `_lockedAmount` for a given buyer should be capped at the remaining principal of that buyer.

Ideally, this case should never happen if protocol restricts `buyProtection` to a condition that cumulative protection bought across all active protections for a single buyer should never exceed the remaining principal for that buyer.

But since no such check exists currently, an additional check to cap `_lockedAmount` per buyer is necessary to prevent loss to sellers

# H06 - Existing buyer who has been regularly renewing protection will be denied renewal even when she is well within the renewal grace period

## Summary

Existing buyers have an opportunity to renew their protection within grace period. If lending state update happens from `Active` to `LateWithinGracePeriod` just 1 second after a buyer's protection expires, protocol denies buyer an opportunity even when she is well within the grace period.

Since defaults are not sudden and an `Active` loan first transitions into `LateWithinGracePeriod`, it is unfair to deny an existing buyer an opportunity to renew (its alright if a new protection buyer is DOSed). This is especially so because a late loan can become `active` again in future (or move to `default`, but both possibilities exist at this stage).

All previous protection payments are a total loss for a buyer when she is denied renewal at the first sign of danger.

## Vulnerability Detail

`renewProtection` first calls `verifyBuyerCanRenewProtection` that checks if the user requesting renewal holds same NFT id on same lending pool address & that the current request is within grace period defined by protocol.

Once successfully verified, `renewProtection` calls [`_verifyAndCreateProtection`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L189) to renew protection. This is the same function that gets called when a new protection is created.

Notice that this function calls `_verifyLendingPoolIsActive` as part of its verification before creating new protection - this check denies protection on loans that are in `LateWithinGracePeriod` or `Late` phase (see snippet below).

```solidity
function _verifyLendingPoolIsActive(
    IDefaultStateManager defaultStateManager,
    address _protectionPoolAddress,
    address _lendingPoolAddress
  ) internal view {
    LendingPoolStatus poolStatus = defaultStateManager.getLendingPoolStatus(
      _protectionPoolAddress,
      _lendingPoolAddress
    );

    ...
    if (
      poolStatus == LendingPoolStatus.LateWithinGracePeriod ||
      poolStatus == LendingPoolStatus.Late
    ) {
      revert IProtectionPool.LendingPoolHasLatePayment(_lendingPoolAddress);
    }
    ...
}

```

## Impact

User who has been regularly renewing protection and paying premium to protect against a future loss event will be denied that very protection when she most needs it.

If existing user is denied renewal, she can never get back in (unless the lending pool becomes active again). All her previous payments were a total loss for her.

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L189

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/libraries/ProtectionPoolHelper.sol#L407

## Recommendation

When a user is calling `renewProtection`, a different implementation of `verifyLendingPoolIsActive` is needed that allows a user to renew even when lending pool status is `LateWithinGracePeriod` or `Late`.

Recommend using `verifyLendingPoolIsActiveForRenewal` function in renewal flow as shown below

```solidity
  function verifyLendingPoolIsActiveForRenewal(
    IDefaultStateManager defaultStateManager,
    address _protectionPoolAddress,
    address _lendingPoolAddress
  ) internal view {
    LendingPoolStatus poolStatus = defaultStateManager.getLendingPoolStatus(
      _protectionPoolAddress,
      _lendingPoolAddress
    );

    if (poolStatus == LendingPoolStatus.NotSupported) {
      revert IProtectionPool.LendingPoolNotSupported(_lendingPoolAddress);
    }
    //------ audit - this section needs to be commented-----//
    //if (
    //  poolStatus == LendingPoolStatus.LateWithinGracePeriod ||
    //  poolStatus == LendingPoolStatus.Late
    //) {
    //  revert IProtectionPool.LendingPoolHasLatePayment(_lendingPoolAddress);
    //}
    // ---------------------------------------------------------//

    if (poolStatus == LendingPoolStatus.Expired) {
      revert IProtectionPool.LendingPoolExpired(_lendingPoolAddress);
    }

    if (poolStatus == LendingPoolStatus.Defaulted) {
      revert IProtectionPool.LendingPoolDefaulted(_lendingPoolAddress);
    }
  }
```

# H07 - User can bypass the `ProtectionPurchaseLimitTimestamp` restriction that disallows protection purchase on a specific lending pool after specific time

## Summary

Protocol imposes a purchase deadline that disallows protection buyers from adding new protection beyond this deadline. Only exception to this rule is old protection buyers who are renewing their purchase - this loophole can be exploited by a malicious user to bypass the time limit.

A protection buyer can start by buying a negligible amount of protection & keep renewing such negligible amount until time limit expires. After expiry, a protection buyer can always increase the protection amount when executing a fresh renewal request. There is no check currently to verify if the protection amount on expired protection is the same as the protection amount in renewal request.

## Vulnerability Detail

When creating a new protection purchase, [`canBuyProtection`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ReferenceLendingPools.sol#L153) verifies if current timestamp exceeds the `lendingPoolInfo.protectionPurchaseLimitTimestamp`. Only exception to this check is when an existing user is renewing protection

```solidity
  function canBuyProtection(
    address _buyer,
    ProtectionPurchaseParams calldata _purchaseParams,
    bool _isRenewal
  )
  {
    ...
        /// When buyer is not renewing the existing protection and
    /// the protection purchase is NOT within purchase limit duration after
    /// a lending pool added, the buyer cannot purchase protection.
    /// i.e. if the purchase limit is 90 days, the buyer cannot purchase protection
    /// after 90 days of lending pool added to the basket
    if (
      !_isRenewal &&
      block.timestamp > lendingPoolInfo.protectionPurchaseLimitTimestamp
    ) {
      return false;
    }
    ...
  }
```

[`verifyBuyerCanRenewProtection`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/libraries/ProtectionPoolHelper.sol#L384) function in `ProtectionHelper` library is responsible for checking if existing buyer holds the same NFT token for same lending pool as expired protection. Note that there is no check in this function to verify if the new requested protection amount is equal (or less than) protection amount of expired protection.

```solidity

  function verifyBuyerCanRenewProtection(
    mapping(address => ProtectionBuyerAccount) storage protectionBuyerAccounts,
    ProtectionInfo[] storage protectionInfos,
    ProtectionPurchaseParams calldata _protectionPurchaseParams,
    uint256 _renewalGracePeriodInSeconds
  ) external view {
    ...
    if (
      expiredProtectionPurchaseParams.lendingPoolAddress ==
      _protectionPurchaseParams.lendingPoolAddress &&
      expiredProtectionPurchaseParams.nftLpTokenId ==
      _protectionPurchaseParams.nftLpTokenId
    ) {
      /// If we are NOT within grace period after the protection is expired, then revert
      if (
        block.timestamp >
        (expiredProtectionInfo.startTimestamp +
          expiredProtectionPurchaseParams.protectionDurationInSeconds +
          _renewalGracePeriodInSeconds)
      ) {
        revert IProtectionPool.CanNotRenewProtectionAfterGracePeriod();
      }
    }
    ...
    }

```

Combining above 2, a malicious user can start off with a very small amount of protection and keep renewing the same within grace period. And when the time comes, she can always increase protection amount. This way a buyer bypasses any time limit restriction imposed by protocol

## Impact

A protection buyer should not be allowed to increase his protection on renewal - at best, she should be able to continue with same protection as before. By starting small, a protection buyer retains an option to increase protection amount well beyond purchase time limit. Such limit exists to protect protection sellers - bypassing this limit would mean potential losses to protection sellers

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ReferenceLendingPools.sol#L153

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/libraries/ProtectionPoolHelper.sol#L384

## Recommendation

Inside the `verifyBuyerCanRenewProtection` , in addition to checking `lendingPoolAddress` and `nftLpTokenId`, I recommend that an additional check to be done to verify that protection amount for renewal request is the same (or less than) protection amount of expired protection.

Recommend following change to `if` condition on line 384

```solidity
    if (
      expiredProtectionPurchaseParams.lendingPoolAddress ==
      _protectionPurchaseParams.lendingPoolAddress &&
      expiredProtectionPurchaseParams.nftLpTokenId ==
      _protectionPurchaseParams.nftLpTokenId &&
      expiredProtectionPurchaseParams.protectionAmount >=
      _protectionPurchaseParams.protectionAmount        //---audit - add this condition on protection amount
    ) {
        ...
    }
```

# H08 - Protection sellers can front-run accrued premium updates and make instant arbitrage profits (assuming sToken has a secondary market)

## Summary

Accrued premium for lending pools is updated via a `cron job` after every payment on underlying lending pool. Accrual increases the `totalSTokenUnderlying`, ie. total underlying backed by `sTokens`. This inturn increases the exchange rate of `sTokens`. All things equal, an accrue premium transaction increases the price of `sTokens`, ie each token held by a protection seller can be exchanged for a bigger amount of underlying tokens

A protection seller can track payment dates of `GoldFinch` lending pools & front-run any cron operation that accrues premium right after payment date. By selling protection right before the `cron` operation in exchange for `sTokens`, a seller is locking in a lower exchange rate for her underlying deposit. Same underlying will fetch her higher number of `sTokens` than what she would get post premium accrual.

Since whitepaper indicates that there will be a secondary market for `sTokens` on Uniswap, such seller can immediately sell these tokens after exchange rate increases post cron job. Repeatedly doing this effectively makes protection seller earn arbitrage profits. Such profits are at the expense of existing protection sellers

## Vulnerability

Technical docs and natspec for [`_accruePremiumAndExpireProtections`]() confirms that a cron job is run right after every payment date.

```solidity
/**
   * @dev Accrue premium for all active protections and mark expired protections for the specified lending pool.
   * Premium is only accrued when the lending pool has a new payment.

   function _accruePremiumAndExpireProtections(
    LendingPoolDetail storage lendingPoolDetail,
    uint256 _lastPremiumAccrualTimestamp,
    uint256 _latestPaymentTimestamp
  )   internal
    returns (
      uint256 _accruedPremiumForLendingPool,
      uint256 _totalProtectionRemoved
    )
  {
    ...
    }
**/
```

Function loops through each lending pool & checks if latest payment timestamp is updated. If yes, then premium is accrued and added to the `totalSTokenUnderlying` in [`accruePremiumAndExpireProtections`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L327)

```solidity
  function accruePremiumAndExpireProtections(address[] memory _lendingPools)
    external
    override
  {
    ...

    if (_totalPremiumAccrued > 0) {
      totalPremiumAccrued += _totalPremiumAccrued;
      totalSTokenUnderlying += _totalPremiumAccrued;
    }
    ...

  }
```

Increasing the `totalSTokenUnderlying` increases the exchange rate of `sTokens` calculated in the [`_getExchangeRate` function](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L913)

```solidity

  function _getExchangeRate() internal view returns (uint256) {
    ...
    uint256 _exchangeRate = (_totalScaledCapital *
      Constants.SCALE_18_DECIMALS) / _totalSTokenSupply; //- audit _totalScaledCapital is totalSTokenUnderlying converted to 18 decimals
    ...
}
```

When a new seller [deposits](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L1035) underlying token, she receives shares based on prevailing `exchangeRate`.

```solidity
  function _deposit(uint256 _underlyingAmount, address _receiver) internal {
    ...
        uint256 _sTokenShares = convertToSToken(_underlyingAmount);
    ...
    }
```

```solidity
  function convertToSToken(uint256 _underlyingAmount)
    public
    view
    override
    returns (uint256)
  {

    ...
       return
      (_scaledUnderlyingAmt * Constants.SCALE_18_DECIMALS) / _getExchangeRate();
  }
```

Note that a lower exchange rate leads to a higher number of shares.

User can easily calculate the next payment date by combining `getPaymentPeriodInDays` and `getLatestPaymentTimestamp` functions in `GoldfinchAdapter`. On this day, seller can front run the `accruePremiumAndExpireProtection` cron job and sell protection at a cheaper exchange rate. Immediately after cron executes, price re-adjusts upwards. Seller can then go to secondary market on Uniswap to sell the tokens to earn instant arbitrage profit

Note that white paper indicates presence of secondary market on Uniswap. Here is an excerpt

_"If sellers wish to redeem their capital and interest before the lockup period, they might be able to find a buyer of their SToken in a secondary market like Uniswap"_

## Impact

There is no free lunch - the free profits made by a seller by front-running is borne by the existing protection sellers. Had such an attack not been executed, the accrual profits would have been distributed to all sellers proportionately.

## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L327

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L913

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L1035

## Recommendation

Key vulnerability here is that premium accrual is discrete & not continuous. Having a `cron` job discreetly update accrued premium exposes the protocol to front-running risks.

This design choice is perplexing as premium accrues to sellers regardless of what happens on payment date. Underlying loan payments don't matter to sellers as their source of yield is NOT the underlying loan payment but the premium already paid by protection buyers. Whether loan defaults or pays full amount, premium sellers will anyway accrue premium based on existing buyer commitments.

Premium accrual should have been automatic & a variable called `premiumAccruedPerSecond` can be introduced that gets updated each time a new protection is bought/renewed (increases) or expired (decreases). Multiplying this accrual rate with time elapsed should automatically give the accrued premium while calculating exchange rate.

This is similar to staking rewards (eg. Synthetix model) that auto accrue based on a pre-determined accrual rate per second.

# H09 - Protection seller can bypass the withdrawal cycle restriction by placing withdrawal requests in advance

## Summary

`requestWithdrawal` allows a protection seller to first place a request for withdrawal. Actual withdrawal is allowed in the `open` state that comes after 1 full pool cycle is completed. This restriction is imposed so that sellers don't exit enmasse when underlying loan pool borrowers show signs of weakness (or market rumors around borrower credit risk).

Protocol also mentions that a secondary market for `sTokens` on Uniswap will exist. Users who don't want to lock up their capital can swap `sTokens` on Uniswap. However, this creates a loophole. A malicious seller can flashswap `sTokens`, place a withdraw request and pay back the `sTokens` immediately. Now that a withdrawal request is placed in advance, users can make the actual deposit anytime in the next cycle & exit much faster.

Protocol is tricked into thinking that the request is genuine. In effect sellers can deposit and withdraw at will. Cost of executing this attack is just the gas fees but the benefit is huge - a protection seller can jump the queue if loans start going bad and exit before others can.

## Vulnerability

[`_requestWithdrawal`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L1088) checks user balance and places a request that is redeemable 2 cycles later.

```solidity
function _requestWithdrawal(uint256 _sTokenAmount) internal {
    ...
   unchecked {
      /// Update total requested withdrawal amount for the cycle considering existing requested amount
      if (_oldRequestAmount > _sTokenAmount) {
        withdrawalCycle.totalSTokenRequested -= (_oldRequestAmount -
          _sTokenAmount);
      } else {
        withdrawalCycle.totalSTokenRequested += (_sTokenAmount -
          _oldRequestAmount);
      }
    }
    ...
    }
```

Whitepaper suggests the existence of secondary market in Uniswap.

_"If sellers wish to redeem their capital and interest before the lockup period, they might be able to find a buyer of their SToken in a secondary market like Uniswap"_

Combining the above 2, seller can do a flashswap to receive `sTokens` and place a withdraw request - once request placed, seller will return `sTokens` to pool. Seller does not have any intent to deposit capital in current cycle.

In the next cycle, seller buys actual protection and receives `sTokens`. Then when that cycle ends and pool agains goes into `Open` state, seller can call [`withdraw`](https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L232) function

Note that there is no check on withdraw function to verify if the user indeed existed 2 cycles before. Only verification is if amount request > withdrawal request

```solidity
  function withdraw(uint256 _sTokenWithdrawalAmount, address _receiver)
    external
    override
    whenPoolIsOpen
    whenNotPaused
    nonReentrant
  {
    ...
    /// Step 2: Verify withdrawal request exists in this withdrawal cycle for the user
    uint256 _sTokenRequested = withdrawalCycle.withdrawalRequests[msg.sender];
    if (_sTokenRequested == 0) {
      revert NoWithdrawalRequested(msg.sender, _currentCycleIndex);
    }

    /// Step 3: Verify that withdrawal amount is not more than the requested amount.
    if (_sTokenWithdrawalAmount > _sTokenRequested) {
      revert WithdrawalHigherThanRequested(msg.sender, _sTokenRequested);
    }
    ...
}

```

## Impact
Sellers can place advance withdrawal requests & then deposit/withdraw at will without respecting the lockup period. Malicious seller can jump the queue of withdrawal requests placed by genuine sellers. 

In times of borrower distress, `exchangeRate` will drop as `lockedCapital` keeps increasing - in those times, faster exit directly translates to lesser losses. 

If someone is allowed to exit the queue earlier, then that opportunity cost will directly result in losses to genuine sellers who lost their turn.


## Code Snippet

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L1088

https://github.com/sherlock-audit/2023-02-carapace-0kage-eth/blob/59cc8acfffbd1431835e9e4ad6483e0e35b70518/contracts/core/pool/ProtectionPool.sol#L232

## Recommendation

The withdrawal design currently implemented is inconsistent with a freely trading secondary market on Uniswap. One possible solution could have been to snapshot `sTokens` multiple times within a given pool cycle - but a Uniswap LP would mean that some genuine users also would transfer their `sTokens` to liquidity pool & hence lose out when snapshots are taken.

A possible solution could be to offer `wsToken`, a ERC1155 wrapper around current `sTokens`, to protection sellers on deposit. `sToken` quantity, exchange rate calculations etc remain unchanged. Only advantage is that `wsStoken` ownership can be clearly established enabling a better control on token withdrawal lockups. 

Maybe dev team & my fellow auditors can come up with better solutions - haven't thought through deep enough.