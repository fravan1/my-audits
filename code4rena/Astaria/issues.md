

2. Gas Optimization

Since epoch check is fundamental for withdrawals & is independent of other condiserations, place this error check at the top

```
 function _redeemFutureEpoch(
    VaultData storage s,
    uint256 shares,
    address receiver,
    address owner,
    uint64 epoch
  ) internal virtual returns (uint256 assets) {
    // check to ensure that the requested epoch is not in the past

    ERC20Data storage es = _loadERC20Slot(); //-n gets stored slot for erc20

    if (msg.sender != owner) {
      uint256 allowed = es.allowance[owner][msg.sender]; // Saves gas for limited approvals.
      // -n gets allownace for msg.sender given by owner

      if (allowed != type(uint256).max) {
        es.allowance[owner][msg.sender] = allowed - shares; //-n reduce allowance by number of shares to redeem
      }
    }

    if (epoch < s.currentEpoch) {
      revert InvalidState(InvalidStates.EPOCH_TOO_LOW); //-n not allowed to reddeem
      //-n -qa save gas - this should be first check
    }
    ....

}
```

3. Vulnerability

`Deposit` function checks shares against minDepositAmount instead of `assets`

```
  function deposit(uint256 assets, address receiver)
    public
    virtual
    returns (uint256 shares)
  {
    // Check for rounding error since we round down in previewDeposit.
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    require(shares > minDepositAmount(), "VALUE_TOO_SMALL"); //-n -q is this shares or assets???
    // Need to transfer before minting or ERC777s could reenter.
    ERC20(asset()).safeTransferFrom(msg.sender, address(this), assets);

    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);

    afterDeposit(assets, shares);
  }
```

4. Gas Optimization

`totalAssets` is being called in deposit without any purpose..

```
 function deposit(
    uint256 amount,
    address receiver
  ) public override(ERC4626Cloned) whenNotPaused returns (uint256) {
    VIData storage s = _loadVISlot();
    if (s.allowListEnabled) {
      require(s.allowList[receiver]);
    }

    uint256 assets = totalAssets(); //-n implied value of vault at current time
    //-n -q what is the purpose of this call?? -> its not being used

    return super.deposit(amount, receiver);
  }

```

5. QA

Wrong Natspec definition for validateLien

Says - Remove all liens for a given collateral token

```
  /**
   * @notice Removes all liens for a given CollateralToken. //-n -qa wrong definition for function
   * @param lien The Lien.
   * @return lienId The lienId of the requested Lien, if valid (otherwise, reverts).
   */
  function validateLien(
    Lien calldata lien
  ) external view returns (uint256 lienId); //

```

Wrong Natspec definition

```
/\*\*

- @notice Removes all liens for a given CollateralToken.
- @param stack The Lien stack
- @return the amount owed in uint192 at the current block.timestamp
  \*/
  function getOwed(Stack calldata stack) external view returns (uint88);
```

```
 /**
   * @notice Removes all liens for a given CollateralToken.
   * @param stack The Lien
   * @param timestamp the timestamp you want to inquire about
   * @return the amount owed in uint192
   */
  function getOwed(
    Stack calldata stack,
    uint256 timestamp
  ) external view returns (uint88);
```

```
  /**
   * @notice Retrieves a lienCount for specific collateral
   * @param collateralId the Lien to compute a point for
   */
  function getCollateralState(
    uint256 collateralId
  ) external view returns (bytes32);
```

6. QA

Does not check if stack is valid -> user can pass any stack and get
the accured interest for vault -> any dashboard can be created

7. Gas Optimization

In `isValidRefinance` in `AstariaRouter`

- first check if collateral Id matches - since this is an independent check -> incase wrong ID is sent -> saves gas on loading storage slot

8.  GetBuyOut function gives a buyout value for a non-existent lien on vault

does not check if stack is part of vault or not

```
function getBuyout(
   Stack calldata stack
 ) public view returns (uint256 owed, uint256 buyout) {
   return _getBuyout(_loadLienStorageSlot(), stack);
 }
```

9. Vault fee is being double charged if borrower makes part payment to lien

- total interest owed is used for calculating vault fees
- if part of payment is remaining 0> again vault fee is charged on balance of interest owed
(already fully charged )


10. `withdraw` function in Public Vault -> uses ERC4626clone implementation

  1. Approval is given to `amount` instead of `maxShares`
  2. In `ERC4626RouterBase` approval is not reset back to 0 after successful withdrawal -> 

11. `convertToShares` and `previewMint` have different behavior at assets = 0 (same with convertToAsset) -> 


12. `flashAction` securityHook check is redundant -> calculating
`preTransferState` and 