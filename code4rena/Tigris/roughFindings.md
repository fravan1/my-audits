1. _MEDIUM RISK_

When `baseFundingRate` is updated using the `setAssetBaseFundingRate` function, an explicit check to ensure `baseFundingRate` is less than `maxBaseFundingRate` is performed in line <>

However no such check is present in the `addAsset` function -> implying that an owner can add an asset with a `baseFundingRate` higher than `maxRate`

_Recommendation_
Add the condition inside the `addAsset` function

2.  _MEDIUM RISK_

_Issue_
When owner sets `maxBaseFundingRate` in `setMaxBaseFundingRate` and decreases maxBaseFundingRate -> there is a possibility that one or more of existing assets have a baseFundingRate > maxBaseFundingRate

When setting maxBaseFundingRate to a new value, there is no check to see if baseFundingRate of all existing assets is below this rate

This could lead to a scenario where the protocol defined maxBaseFundingRate is less than actual baseFundingRates used by 1 or more assets

_Proof of Concept_

Bob is a protocol owner who deployed contract with a `maxBaseFundingRate` set at its default value of `1e10`. Bob deploys an asset `BTC/USDC` against an index 5 with a baseFundingRate of `5e9`. Later Bob calls the `setMaxBaseFundingRate` function to set `maxBaseFundingRate` to `1e9`. We end up in a state where the `maxBaseFundingRate` < `BTC/USDC` basefundingRate

_Recommendation_
Run a loop offchain over all earlier pair indices (known in advance) to check if baseFundingRate > `setMaxBaseFundingRate` -> if yes, then call the `setAssetBaseFundingRate` for that index with the `maxBaseFundingRate` as input to readjust funding rates inline with max rates

4. Missing check when subtracting `amount` from `modifyLongOi` and `modifyShortOi`.

Function reverts If `_onOpen` is `false` and `amount` passes is greater than `_idToOi[_asset][_tigAsset].longOi` in `modifyLongOi`. Same applies when `amount` passed is greater than `_idToOi[_asset][_tigAsset].shortOi` in `modifyShortOi`

_Recommendation_

Make a validation check

5. `modifyMargin` function changes `margin` and `leverage`. However this does not update the `initId[id]` calculation -> which explictly takes in margin and leverage

When I compare other functions `addToPosition` and `reducePosition` -> I notice that `initId` is getting recalculated when the margin changes with position.

_Impact_

Since `initId[]` is being used in calculating accruedinterest 0> an incorrect value will adversely impact the caplculations - marked it as high

_Recommendation_

Since its the same calculation, abstract it into a private function -> and use the function when margin and/or leverage are updated

6. `burn` function does not delete `init[id]` for the specific id -> Medium risk -> while all the relevant mappings related to `trade` are deleted -> `init[id]` is not deleted -> this can lead to an undefined behavior

_Recommendation_

Delete `init[id]`

7. `setAllowedAsset` can be set to `true` for any address by owner without checking if such address is indeed added to the `assets` list-> There is no validation here - this could result in accidental approvals by owners

_Recommendation_

Check explicitly if that address indeed is in the list

```
    require(assets[assetsIndex[_asset]] == _asset, "Does not exist");
```

8. Any user can call the `distribute` function in governance contract `govtNFT` -> and if a user has `tgUSD` token and accidentally approves tokens -> tokens will move from user account to governance account

This can be dangerous -> as the contract name `distribute` might be misinterpreted by a user who may think that this function distributes rewards to user - this function, as I understand, is used by `Trading` contract to send rewards to governance contract which inturn need to be claimed by users by calling `claim` function

_Recommendation_
This transfer of tokens might be difficult to recover - once transferred. Since this function is to be strictly called by trading -> avoid access and define a `onlyProtocol` modifier like the one defined in `PairsContract`

9. `Claim` function allows users to claim rewards as owners of govNFTS

does not check if amount > 0 -> any person who does not own a governance NFT also has a potential to re-enter the txn via the `transfer` function

\***_ This one I'm dropping -> reentrancy risk does not exist as earlier thought _**

10. _feePaid_ returned by the function `_handleOpenFees` fails to include the referral fee component

In the `_handleOpenFees`, _feePaid_ returned by the function is calculated in formula in line 773 (give reference)

Note that `daoFeePaid` variable `= _positionSize * _fees.daoFees / DIVISION_CONSTANT;`

`_fees.daoFees` has already accounted for removal of `2*_fees.referralFees` in lin 754 & `_fees.botFees` in line 765

- So the `daoFeePaid` only includes fees excluding 2\*referral and botfees
- So when `feePaid` is calculaed in formula on line 773 -> we are adding back burn fees and bot fees to the `daoFeesPaid` but missed the `referralFee` component that we earlier subtracted from `daoFeePaid`

As a result the value returned by the `handleOpenFees` is incorrect

formula should be replaced by below

```
   _feePaid =
                _positionSize
                * (_fees.burnFees + _fees.botFees + 2* _fees.referralFees)
                / DIVISION_CONSTANT // divide by 100%
                + _daoFeesPaid;
```

11. Unlike `referral` and `botFees` which are minted into the trading contract, `burnFees` is not being accounted furn.

- Missing `burnFrom` method to account for the `burnFees` if \_fee.burnFees > 0
- As I understand, `burnFees` component of fee needs to be burnt as per design - This woulg mean `IStable(_tigAsset).burnFrom()` function needs to be called to burn the corresponding fees, if non-zero. There is no such function call in the \_handleOpenFees - Same issue also appears in the \_handleCloseFees function

12. In the `_handleCloseFees` return variable of the function `_payout` incorrectly accounts for referral fee

```
        payout_ = _payout - _daoFeesPaid - _burnFeesPaid - _botFeesPaid;
```

Above formula can also be written as

payout\_ = \_payout - (\_daoFeesPaid + burnFeesPaid + botFeesPaid)

Note that `_daoFeesPaid` variable in above formula has already accounted for

- `2*_referralFeesPaid` that is already subtracted
- `_botFeesPaid` that is also subtracted

We added back _botFeesPaid to `daoFeesPaid` but the 2\*\_referralPaid is not added back tot he formula -> so the payout_ computed will allways be higher than the actual value

## Missing Validations

7. Many functions have missing input validations, resulting in potentially unsafe usage -> I've listed all such validations in this one issue, along with recommendations
   7.1

```
   function openPositionsSelection(uint _from, uint _to) external view returns (uint[] memory) {
```

- check that to >= from
- \_to should be < openpositions.length // anything above will be undefined

  7.2

```
    setMinter(_minter, )
```

- minter cannot be 0 address
- minter has to be trade contract - how to ensure this

  7.3

```
   function trades(uint _id) public view returns (Trade memory)
```

    -   Make sure _id returns a valid trade

## Gas Optimizationa

1. `modifyLongOI`

Storage is expensive -> store to memory

memory OpenInterest oi = \_idToOi[\_asset][_tigasset]
uint256 longOi = oi.longOI;
uint256 maxOi = oi.maxOI

````
    function modifyLongOi(uint256 _asset, address _tigAsset, bool _onOpen, uint256 _amount) external onlyProtocol {
        if (_onOpen) {
            longOi += _amount;
            require(longOi <= maxOi || maxOi == 0, "MaxLongOi");
        }
        else {
            longOi -= _amount;
            if (longOi < 1e9) {
                _idToOi[_asset][_tigAsset].longOi = 0;
            }
    ```
````

2. In `GovNFT`

```
    function _burn(uint256 tokenId) internal override {
        }
```

- No validation for valid tokenId
- check if tokenId exists

3.
