### Notes

- Protocol mints a stable coin that allows users to exchange 1 UXD for 1 USD worth of assets in collateral
- Perp protocol depository is being used
- Protocol maintains delta neutral positions by either quote mint/redeem or rebalancing
- All decisions go through governance
- UXDController is key user interfacing contract for minting and burning UXD token
- PositiveP/L disabled

### Logic

- `UXDController`

  - `mint`

    - takes in `assetToken`, `assetAmount` (collateral), `minAmountOut` (min UXD tokens to be minted), `receiver` - address that will get the tokens

    - gets the asset collateral (ERC 20) from asset token address
    - checks if user has given approval for the amount being passed
    - find depository address for the given `assetToken`
      - note that this function `findDepositoryForDeposit` takes the first depository address registered for the given token (it will always be the same, regardless of how many assets added)
    - transfer collateral asset for `assetAmount` from user to depository
    - Builds an internal struct `InternalMintParams` which passes asset, amount, depository, minAmountOut, receiver to `_mint` function
    - `_mint`
      -

### Tests

**Mocks**

- Review Mocks

  - `TestWETH9` - ok
  - `TestERC20` - 1 million supply - ok
  - `TestDepository` -ok

    - depository - without external interactions
    - `deposit` and `redeem` functions
    - mainly accounting entries - does not have any external calls

  - `MockController` - ok

    - initializes a depository (I'm assuming mock depository)
    - test `deposit` and `withdraw`
    - tests `updateDepository` and `setRedeemable`

  - `MockPerpAccountBalance` - ok

    - mock contract that gives 0 values for all account positions
    - external contract as this is Perp protocol contract
    - Mocking it to set values we want to check functionality in out contracts
    - Following values

      - `_openNotional` - for each account -> market
      - `_absPositionValue` - for each account
      - `_pnlAndPendingFee` - for each account
      - `_debtValue` - for each account
      - `_totalPositionValue` - total value of position
      - `_positionSize` - total size
      - `_vault` - vault address

      - `getVault`
      - `setOpenNotional`
      - `getTotalOpenNotional`
      - `setTotalPositionValue`
      - `getTotalPositionValue`
      - `setTotalPositionSize`
      - `getTotalPositionSize`
      - `getTotalDebtValue`
      - `setTotalDebtValue`
      - `setPnlAndPendingFee`
      - `getPnlAndPendingFee`
      - `setTotalAbsPositionValue`
      - `getTotalAbsPositionValue`

- `MockPerpClearingHouse` - ok

  - Mock contract to mock functionality of Perp protocol clearing house
  - Cotains following
    - accountBalance
    - exchange address
    - accountValue
    - `setAccountBalance`
    - `setExchange`
    - `getExchange`
    - `getAccountBalance`
    - `openPosition` - returns event and base/quote amounts

- `MockPerpExchange` - ok

  - Mocks exchange contract of Perpetual Protocol
  - has 2 functions
    - `getAllPendingFundingPayment` - hardcodes to 100 ether
    - `getSqrtMarkTwapX96` - returns 0

- `MockPerpMarketRegistry`

  - Mocks perp protocol market registry
  - implements 1 function
    - `getFeeRatio` - hardcodes to 10 \* 1e6

- `MockPerpVault` - ok

  - mocks the perp vault for Perp protocol
  - has following state variables
    - `balance` - total balance in quote token
    - `_freeCollateral` - account -> amount mapping
    - `_freeCollateralByToken` - account -> asset token -> amount mappiong
    - `_balanceByToken` - account -> asset token -> balance mapping
  - also implements functions
    - `deposit`
    - `withdraw`
    - `getBalance`
    - `setFreeCollateral`
    - `getFreeCollateral`
    - `setFreeCollateralByToken`
    - `getFreeCollateralByToken`
    - `setBalanceByToken`
    - `getBalanceByToken`

- `MockUniswapRouter` - ok

- uniswap router for swapping assets
- implements `exactInputSingle` function to swap one asset to another

- `TestPerpDepositoryUpgrade` - ok

  - PerpDepository is a UUPS upgradeable contract
  - implements `VERSION` to check if contract is upgradeable

- `TestUXDControllerUpgrade` - ok

- UXD controller is a UUPS upgradeable contract
- implements `VERSION` to check if contract is upgradeable

**Fixtures**

- `ControllerFixture` - ok

  - defines `account[0]` as admin
  - deploys `TestWeth9`
  - deploys Redeemable token(UXD)- `TestERC20` -> `RED` is token symbol
  - deploys asset token - `TestERC20` -> `ASS` is token symbol
  - deploys `UXDController` - with `weth` address defined above
  - deploys `UXDRouter`
  - deploys `TestDepository` - test depository (mock)
  - registers `asset token` to `depository` address using `UXDRouter` defined above
  - registeras `weth token` to `depository` address using `UXDRouter`
  - whitelist `asset token` to be used as collateral -> done by controller
  - whitelist `weth token` to be used as collateral -> done by controller
  - update `router` address inside controller using `updateRouter`
  - set redeemable token inside controller using `setRedeemable`
  - transfer 100 `ASS` tokens to controller
  - transfer 100 `WETH` tokens to controller
  - returns `controller`, `router`, `depository`, `weth`, `asset`, `redeemable`

- `DepositoryFixture` - ok

  - defines `account[0]` as admin
  - get `TestERC20` contract
  - deploy `Redeemable` token which is `TestERC20` contract - `RED` symbol
  - deploy `MockController` -> call `setRedeemable` to set `RED` token
  - deploy `MockVault` - mock vault of Perp protocol
  - deploy `MockAccountBalance` - mock account balance contract of Perp protocol
  - deploy `MockPerpExchange` - mock exchange contract of Perp protocol
  - deploy `ClearingHouse` - mock clearing house contract of Perp protocol
  - set account balance address of of `ClearingHouse` to `MockAccountBalance` defined above
  - set exchange address of `ClearingHouse` to `MockExchange` defined above
  - deploy `MockRegistry` - mock registry contract of Perp protocol
  - deploy `TestContract` - empty contract basically
  - deploy `Quote` token - `QTE`ERC20 token using `TestERC20` contract
  - deploy `Collateral` token - `COL` ERC20 token using `TestERC20` contract
  - deploy `UUPS` Proxy contract (implementation also gets deployed simultaneoulsy) for `PerpDepository` contract - note that this is NOT a mock contract -> but the addresses used are all mock addresses
  - update controller depository address with depository contract above
  - set redeemable soft cap in controller `1 million` ether
  - return `controller`, `vault`, `depository`, `quoteToken`, `baseToken`, `accountBalance`

- `RageDepositoryFixture` - TBD

**Tests**

- ## `UXDControllerTests`

### Questions

1. How does rebalancing work when there is a huge volatility in ETH price?

2. Who does rebalancing and when is rebalancing triggered?

3. Who can do Quote mint? And when is it done?

4. Single sided rebalancing - how will it ensure delta neutrality?
