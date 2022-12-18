# 1. Calling `modifyMargin` causes incorrect `accrued interest` & `P/L` computation for traders

## Risk Classifcation

High

---

## Lines of Code

[Position.sol Line 197](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Position.sol#L197)

[Position.sol Line 62](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Position.sol#L62)

[Trading.sol Line 434](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L434)

---

## Vulnerability Details

### Impact

`modifyMargin` function in [Line 197 of Position.sol](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Position.sol#L197) allows a minter the ability to change both `margin` and `leverage` for an existing trade

However, code does not update mapping `initId ` in this function -> `initId` is a function of total position size (which depends on margin and leverage). If you note the other functions in the same contract that change margin, ie `addToPosition` and `reducePosition`, variable `init` is updated within each of these function. Such an update is missing in `modifyMargin` function.

This variable `initId` is used to calculate accrued interest on trade in [Line 62](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Position.sol#L62)

Incorrect calculation on accrued interest directly impacts the final P/L of traders, calculated in [Line 434 of Trading.sol](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L434)

Since this directly relates to P/L earned by platform users, I've marked this issue as HIGH risk

### Proof of Concept

Bob is a trader who has an existing position with margin M and leverage L. Bob increases his Margin to M1 and increases leverage to L1 -> any subsequent accrued interest calculation on that position will overshoot the actual value (`initId` variable is not updated to reflect new leverage and margin and is lower than what it actually should be). As a result, traders Profit will be lesser than actual OR loss will greater than actual.

### Recommended Mitigation Steps

Update the `init` computation in the `modifyMargin` function as shown below

```
function modifyMargin(uint256 _id, uint256 _newMargin, uint256 _newLeverage) external onlyMinter {
    _trades[_id].margin = _newMargin;
    _trades[_id].leverage = _newLeverage;

    initId[_id] = accInterestPerOi[_trades[_id].asset][_trades[_id].tigAsset[_trades[_id].direction]*int256(_newMargin*_newLeverage/1e18)/1e18;
}
```

<br></br>

# 2. Open access to `distribute` function can lead to token loss for users & mess up rewards accounting

## Risk Classifcation

High

---

## Lines of Code

[GovNFTBridged.sol Line 238](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFTBridged.sol#L238)

[Trading.sol Line 749](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L749)

[Trading.sol Line 808](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L808)

---

## Vulnerability Details

### Impact

`distribute` function on [line 238 of GovNFT.sol](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/GovNFTBridged.sol#L238) is responsible to distribute `daoFees` back to NFTHolders. Currently, this function is `external` and hence open to anyone

This function is called by [\_handleOpenFees](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L749) and [\_handleCloseFees](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L808) to distribute rewards.

Creates 2 potential issues:

- Users could accidentally call this function & lose their `tgUSD` in their account. Name `distribute` might be misinterpreted by a user who may think that this function distributes rewards - if a user accidentally approves `tgUSD`, this function transfers `tgUSD` from an unsuspecting user and puts it into the `GovNFT` contract. By exposing a function that should have been strictly accessible only to `Trading` contract to distribute rewards, contract has created unnecessary user vulnerability vector

- When this function is legitimately called by `_handleCloseFees` and `_handleOpenFees` functions in `Trading.sol` -> both functions emit `FeesDistributed` event that has values of dao fees, burn fees, referral fees and bot fees. However, a malicious user can deposit `tgUSD` tokens into `govNFT` contract using this `distribute` function and disrupt the entire accounting logic.

Sum total of all the fees calculated by `event` emissions will not match the fees distributed to govNFT holders. This can create inconsistent states on-chain viz-a-viz front end dashboards.

Since this can adversely impact both new users & protocol itself, I've marked it as a `HIGH` risk finding

### Proof of Concept

Alice is a first time user of Tigris. Alice has tgUSD in her account & also holds governance NFT. Alice interacts with `GovNFT` contract by calling the `distribute` function -> Alice approves `tgUSD` amount and sees that `tgUSD` is transferred out of her account.

### Recommended Mitigation Steps

Since this function is to be strictly called by trading -> avoid providing generic access and define a `onlyProtocol` modifier like the one defined in `PairsContract` where only `trading` contract can access it to distribute rewards

<br></br>

# 3. Burn Fees deducted from trader margin is not burnt in `_handleCloseFees` and `_handleOpenFees`

## Risk Classifcation

High

---

## Lines of Code

[Trading.sol Line 734](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L734)

[Trading.sol Line 805](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L805)

---

## Vulnerability Details

### Impact

Burn fees is deducted from trader margin as per formula in [line 734 of Trading.sol](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L734) of `_handleOpenFees` and in computing net trader payout in [line 805](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L805) of `_handleCloseFees`.

However in both cases, this burn fees is not actually burnt. To burn the fees, dev was supposed to call `burnFrom` function defined in [StableToken](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/StableToken.sol#L13). I don't find a call to this function in either of these 2 implementations (handleOpenFees & handleCloseFees)

This will lead to imbalance in accounting when burn fees > 0 - and since this directly impacts the circulating supply of `tgUSD` token, I have marked it as `HIGH RISK`

### Proof of Concept

Bob is a whale who closes a trade levered 30x and margin of 1m creating a 30m position. Protocol burn fees is 0.01% leading to a burn rate of 300 tgUSD tokens - this is accounted for while paying net amount to Bob but protocol is not burning these 300 tokens creating an imbalance in accounting

### Recommended Mitigation Steps

Use following code to burn tigUSD in both `handleOpenFees` and `handleClosedFees`

```
          IStable(_tigAsset).burnFrom(
              _msgSender(),
              _burnFeesPaid
          );

```

<br></br>

# 4. Adding to an existing position results in leakage of margin in stable vault

## Risk Classifcation

High

---

## Lines of Code

[Trading.sol Line 275](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L275)
[Trading.sol Line 651](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L651)

---

## Vulnerability Details

### Impact

'addToPosition' function adds additional margin to an existing trade. Margin value sent to '\_handleDeposit' function sends margin adjusted for 'fees' instead of sending full margin. This results in a lesser margin withdrawn from traders account - since this directly impacts the margin amount deposited in stable vault, I have assigned 'HiGH' risk to this finding

In [Line 275](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L275), margin value sent to '\_handleDeposit' is '\_add Margin -\_fee' instead of '\_addMargin'.

In [Line 651](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L651), note that the 'margin' amount is transfered from trader to this contract. Since we sent a 'margin adjusted for fees', a lesser margin will be deducted from trader account.

This messes with the fees calculations as tgUSD is minted to govNFT holders with no collateral backing the tokens. On a cumulative basis, when traders add Margin, this shortfall keeps growing

### Proof of Concept

Bob is a trader with an existing position on ETH/USDT pair. Bob adds an additional 100k USDT as margin. At a 0.1% mint fee, Bob will have a fees of 100$. Current contract is only sending 99,900 USDT margin amount to '\_handleDeposit' which subsequently withdraws this amount from trader account.

100$ that was supposed to be rewarded to Gov NFT holders never made it to the protocol. Essentially the tgUSD minted to referrer/ bots is unbacked because of this shortfall

### Recommended Mitigation Steps

Strangely in Line 180 , when a new market order is created, logic is correctly handled. '\_tradeinfo.margin' is sent to '\_handleDeposit' instead of '\_marginAfterfees'. However exact same logic in 'addPosition' is not followed.

I recommend to pass '\_addMargin' instead of '\_addMargin -\_fee' to the 'handleDeposit' function in line 275 for correct accounting

<br></br>

# 5. Funding Rate can exceed maximum allowed `base funding` fate while adding a new asset pair

## Risk Classifcation

Medium

---

## Lines of Code

[PairsContract.sol Line 62](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/PairsContract.sol#L62)

[PairsContract.sol Line 95](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/PairsContract.sol#L95)

---

## Vulnerability Details

### Impact

When adding a new asset (trading pair) using the `addAsset` function in `PairsContract.sol`, base funding rate is set to input valued passed to the function in [Line 62 of PairsContract.sol](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/PairsContract.sol#L62). At this stage, no check is done if the funding rate is below maximum allowed rate.

However, when updating a `basefunding` rate by calling `setAssetBaseFundingRate` in [line 95](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/PairsContract.sol#L95), this check is done.

This logic inconsistency allows for an asset funding rate to exceed the maximum allowed funding rate for protocol. Since funding rate directly impacts the trader P/L, I've marked this issue as `MEDIUM` risk

### Proof of Concept

Bob, protocol owner, incorrectly creates an asset pair with funding rate > Max rate. This asset gets created and when a trader places a trade on this asset, funding fees exceeds the max limit imposed by protocol

### Recommended Mitigation Steps

Validate the funding rate while a new asset is being created.

Replace line 62 below

```
   _idToAsset[_asset].baseFundingRate = _baseFundingRate;
```

with following

```
require(_baseFundingRate <= maxBaseFundingRate, "baseFundingRate too high");
_idToAsset[_asset].baseFundingRate = _baseFundingRate;
```

<br></br>

# 6. Setting `maxBaseFundingRate` could lead to a scenario where funding rate on existing assets > `maxBaseFundingRate`

## Risk Classifcation

Medium

---

## Lines of Code

[PairsContract Line 125](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/PairsContract.sol#L125)

---

## Vulnerability Details

### Impact

`maxBaseFundingRate` variable is supposed to cap the funding rate on all assets in `PairsContract`. There is a provision for owner to change `maxBaseFundingRate` by calling function (setMaxBaseFundingRate)[https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/PairsContract.sol#L125).

However, this function does not check if newly set `maxBaseFundingRate` exceeds the base funding rate of pre-existing assets. This could lead to a situation where some assets have a baseFundingRate higher than max Rate (scenario possible when protocol lowers its maxFundingRate)

Since fundingRate has direct impact on P/L, I've marked this as MEDIUM risk issue

### Proof-of-Concept

Bob is a protocol owner who deployed contract with a `maxBaseFundingRate` set at its default value of `1e10`. Bob deploys an asset `BTC/USDC` against an index 5 with a baseFundingRate of `5e9`. Later Bob calls the `setMaxBaseFundingRate` function to set `maxBaseFundingRate` to `1e9`. We end up in a state where the `maxBaseFundingRate` < `BTC/USDC` basefundingRate

### Recommended Mitigation Steps

Right after setting a new maxBaseFundingRate, in the event that `maxBaseFundingRate` is reduced from earlier value, protocol owner should run a loop off-chain to call function i`dToAsset[assetIndex]` over all earlier pair indices (known in advance) to check if asset.baseFundingRate > `maxBaseFundingRate` -> if yes, then call the `setAssetBaseFundingRate` for that index with the `maxBaseFundingRate` as input to readjust funding rates inline with max rates

<br></br>

# 7. `_handleCloseFees` returns a `_payout` that does not account for referral fees

## Risk Classifcation

Medium

---

## Lines of Code

[Trading.sol Line 805](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L805)

---

## Vulnerability Details

### Impact

Payout calculated on closing a trade accounts for daoFees, burnFees, botFees but misses `referralFees`. This means that payout to trader is higher than what it is supposed to be. Since the error concerns payments to users, I have classified it as `High Risk` finding

Payout is calculated in line 805 as follows

```
  payout_ = _payout - _daoFeesPaid - _burnFeesPaid - _botFeesPaid;
```

Above formula can also be written as

```
payout_ = _payout - (daoFeesPaid + burnFeesPaid + botFeesPaid)
```

Note that `_daoFeesPaid` variable in above formula has already accounted for

- `2*_referralFeesPaid` that is already subtracted
- `_botFeesPaid` that is also subtracted

current formula adds back `_botFeesPaid` to `daoFeesPaid` but the `2*_referralPaid` that was earlier subtracted from `daoFeesPaid` is not added -> so the payout\_ computed will always be higher than the actual value

### Recommended Mitigation Steps

Use formula below for line 805

```
payout_ = _payout - (daoFeesPaid + burnFeesPaid + botFeesPaid + 2* _referralFeesPaid);
```

<br></br>

# 8. QA Report

## Vulnerability Details

### Issue

To close a trade, `burn(id)` function is called in [Line 260 of Position.sol](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Position.sol#L260). All mappings related to the trade ID are deleted except `initId` which stores accrued interest values for the given trade.

This does not materially impact calculations but leads to uneccessary storage overload

### Recommended Mitigation Steps

Add the following immediately after [Line 279](https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Position.sol#L279)

```
delete initId[_id];

```
