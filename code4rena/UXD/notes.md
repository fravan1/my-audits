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

      - checks if asset token is in whitelisted assets list or reverts
      - calls the `deposit` function in depository that returns the `amountOut` of quote token
      - `deposit` in depository

            - goes to `perpDepository` deposit() implementation
            - `deposit` is defined in `IDepository` interface
            - implementation is in each specific depository
            - `deposit` amount takes `asset`, `amount` -> can only be called by `Controller`
            - has implementation if `asset` is base token or quote token
            - right now, from controller -> we will access base token implementation
            - calls `_depositAsset` function
            - `_depositAsset`
                - increases `netAssetDeposits` of deposits by asset amount passed
                - approves `amount` to be accessed by `vault`
                - calls `vault` deposit - external protocol function - lets assume it deposits amount into the vault against given `account` which is depository
            - calls `_openShort` function - whenever we deposit asset, we need to open short position on Perp DEX
            - `_openShort`
                - returns `baseAmount` and `quoteAmount`
                - passes to `_placePerpOrder` with amount, `short`, `exactInput`
                - `_placePerpOrder`
                    - Creates a struct called `OpenPositionParams` as part of IClearingHouse ( this is Perp Protocol contract)
                        - takes `baseToken`,
                        - `baseToQuote` - true (short base)/ false (long base)
                        - `isExactInput` - true -> we specify exact input, false - exact output
                        - `amount` - amount for which perp position is opened
                        - `oppositeAmountBound` - not relevant in our case (0)
                        - `deadline` - set to block.timestamp
                        - `sqrtPriceLimitX96` -similar to uniswap v3
                    - call `clearingHouse.openPosition()` - opens position -> returns `baseAmount` and `quoteAmount` that is the position size (`baseAmount` * `price` at which perp executed)
                    - calculated `feeamount` by passing `quoteAmount` to `_calculatePerpOrderFeeAmount`
                    - `feeamount` is `protocol fee * quote amount`
                    - fee is added to `totalFeesPaid`


                - adds `quoteAmount` to `redeemableUnderManagement`
                - `_checkSoftCap()` checks if quoteAmount > `redeemableSoftCap` assigned for the depository (each depository has max cap attached for redeemable under management)
                - returns `quoteAmount`

    - checks is `quoteAmount` < `minAmount` - reverts if it is
    - mints `quoteAmount` to `receiver`
    - returns `quoteAmount`

- `redeem`

  - takes `assetToken`, `redeemAmount`, `minAmountOut`, `receiver`
  - creates a internal struct `InternalRedeemParams` with above parameters
  - Calls `_redeem(params)`
  - checks whitelisted asset
  - checks allowance for redeeming `redeemAmount` from user
  - finds `depository` to redeem
  - calls `depository.redeem` by passing `asset` and `amountToRedeem`
  - `depository.redeem()`

    - can only be called by controller
    - can handle asset token redeem and quote token redeem (disabled currently)
    - Calls `_openLong()`

      - passes `_placePerpOrder` with `amount`, `isLong`, `isExactInput`
      - `_placePerpOrder`

        - call `clearingHouse.openPosition()` - opens position -> returns `baseAmount` and `quoteAmount` that is the position size (`baseAmount` \* `price` at which perp executed)
        - calculated `feeamount` by passing `quoteAmount` to `_calculatePerpOrderFeeAmount`
        - `feeamount` is `protocol fee * quote amount`
        - fee is added to `totalFeesPaid`

      - subtracts `amount` from `redeemableUnderManagement`

- Call `withdrawAsset`

- reduce `

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

**Manual Tests**

- `Deploy Core`

  -

- ## `UXDControllerTests`

### Deployment order

- `01-deploy-core`

  - checks if Optimism network
  - deploys contracts
    - deploy `UXDController` proxy with `WETH` contract (from config)
    - deploy `UXDRouter`
    - deploy `UXDToken` with `UXDController` address
  - verify contracts
    - verify controller proxy
    - verify controller implementation (get implementation upgrades.erc1967.getImplementationAddress(controller))
    - verify router
    - verify UXD
    - persist artifacts (controller / router / UXD)
  - setup
    - whitelist `WETH` asset in controller
    - whitelist `USDC` asset in controller
    - update router address in controller
    - mintcap set to 2 million UXD tokens
    - set Redeemable token in controller as UXD
    - set UXD local mint cap as 2 million UXD
  - save
    - save `controller`, `router`, `uxd` addresses to `./addresses/optimism/core.json` file

- `02-deploy-perp-depository`

  - get config by `loadConfig(hre)`
  - load Contracts by `loadCoreContracts` - loads all 3 contracts in previous deploy step (controller/router/uxd)
  - `deployContracts`
    - get `PerpDepository` contract
    - deploy using addresses of `PerpVault`, `PerpClearingHouse`, `PerpMarketRegistry`, `PerpVETHMarket`, `WETH`, `USDC` and `UXDController` address in previous step
    - Since it is a proxy contract (uups), we use `upgrades.deployProxy` and `{kind: "uups"}`
    - deploy UniSwapper with uniswap V3 address in config file
  - `verifyContracts`

    - verify depository (why not passing inputs while verifuing)
    - get `depositoryImpl` address using `upgrades.erc1967.getImplementationAddress(depository.address)` to get implementation address
    - verify `depositoryImpl`
    - verify `uniSwapper`

  - `setup`

    - get the `uxdRouter` contract
    - set redeemable softcap in depository (this is the UXD exposure limit in Perp)
    - `setSpotSwapper` - set address of swapper inside depository
    - In the router contract, register new depository with `depository.address` and `WETH` address
    - In the router contract, register new depository with `depository.address` and `USDC` address

  - `save`
    - save depository created in `./addresses/optimism/depositories.json`

- `3_deposit_insurance`

  - get signers - `admin` and `bob`
  - load config parameters - `loadConfig(hre)`
  - get depositories address - `loadDepositories(hre)`
  - get `PerpDepository` instance with contract and address above
  - use `ERC20` contract and initiate a `USDC` instance using USDC address in config
  - `10` USDC -> approve 10 USDC
  - Use `depositInsurance` to deposit `10 USDC` from `admin` address

- `4_mint`

  - get signers - `admin` and `bob`
  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get `UXDController` contract
  - get `UXDToken` contract
  - Attach `WETH9` address to `ERC20` contract
  - Approve `0.01` WETH to be spent by `UXDController`
  - Mint UXD token by calling `mint` function on `UXDController` -> and send it to `config.addresses.Deployer`
  - Check `UXDTotalSupply` by calling `totalSupply` on UXDtoken

- `5_redeem`

  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get signers - `admin` and `bob`
  - get `UXDController` by using controller address
  - get `UXDToken` with uxd token
  - get `totalSupply`
  - call `redeem` on controller with `weth` token, `uxd` token, minAmout =0, deployer address (receiver)

- `6_mint_with_eth`

  - get signers - `admin` and `bob`
  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get `UXDController` contract instance
  - get `UXDToken` contract instance
  - take `eth` amount of 0.01
  - get `totalSupply` of UXD token
  - mint with `eth` - `0` min amount, and address is `deployer`, value is 0.01 eth
  - get new `totalSupply` of UXD token

- `7_redeem_eth`

  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get signers - `admin`
  - get `UXDController` by using controller address
  - get `UXDToken` with uxd token
  - call `redeemForEth` on controller with `uxdAmount`, minAmountOut =0, receiver = `config.addresses.Deployer`
  - get total `uxdSupply`

- `8_withdraw_insurance`

  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get signers - `admin` and `bob`
  - get depositories address - `loadDepositories(hre)`
  - get `PerpDepository` contract instance
  - get `10` USDC
  - withdraw 10 USDC and send to `tokenReceiver` address in config

- `9_deploy_router` **Redundant**

  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get `UXDRouter` instance

- `10_deploy_perp_account_proxy`

  - Get `PerpAccountProxy` contract
  - deploy using `PerpAccountBalance` and `PerpClearingHouse` addresses from `config.contracts`
  - verify contract
  - save address to `addresses/optimism/perpAccountProxy.json`

- `11_quote_mint`

  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get depositories address - `loadDepositories(hre)`
  - get instance of `UXDController`
  - get instance of `PerpDepository`
  - get instance of `uxdToken` and `usdc`
  - approve for `controller` to spend 0.01 usdc
  - get `unrealizedPnL` from depository
  - get total `uxd` supply
  - mint with quote token (`USDC`)-> amount 0.01 usdc, and send to `config.addresses.TokenReceiver`
  - get total `uxd` supply
  - get total `unrealizedPnL`

- `12_quote_redeem`

  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get instance of `UXDController`
  - get instance of `uxdToken`
  - get totalsupply of `uxd`
  - approve spend of `150` uxd
  - redeem 150 `uxd` using `usdc` as token
  - get total `uxd` supply

- `13_rebalance_negative_pnl`

  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get depositories address - `loadDepositories(hre)`
  - get periphery (`perpAccountProxy` contract) - `loadPeriphery()`
  - get instance of `perpDepository`
  - get instance of `perpAccountProxy`
  - get position value by calling `depository.getPositionValue()`
  - get total position size by calling `proxy.getTotalPositionSize` on `perpAccountProxy` - pass it `PerpDEpository` address and `PerpVETHMarket` address
  - take usdc = 4.7, polarity is -1 (negativePL)
  - Get `usdc` contract instance
  - approve 4.7 usdc spend by `depository` address
  - call `rebalanceLite` on `depository` wuth `usdAmount`, `polarity`, `targetPrice` (0) and `admin` address
  - get `positionValue` by calling `depository.getPositionValue()`
  - get `positionSize` by calling `proxy.getTotalPositionSize` on `perpAccountProxy`
  - `getUnrealizedPnL`

- `13_rebalance_positive_pnl`
  - load config parameters - `loadConfig(hre)`
  - load core contracts - `loadCoreContracts(hre)`
  - get depositories address - `loadDepositories(hre)`
  - get periphery (`perpAccountProxy` contract) - `loadPeriphery()`
  - get `admin` signer
  - get instance of `perpDepository`
  - get instance of `perpAccountProxy`
  - get `uxd` token instance
  - get total `uxd` supply - `uxdToken.totalSupply()`
  - get `positionValue` by calling `depository.getPositionValue()`
  - get `positionSize` by calling `proxy.getTotalPositionSize` on `perpAccountProxy`
  - get `WETH` contract instance
  - get `unrealizedPnL` by `depository.getUnrealizedPnl()`
  - approve 0.1 `WETH` spending by `depository`
  -

### Questions

1. How does rebalancing work when there is a huge volatility in ETH price?

2. Who does rebalancing and when is rebalancing triggered?

3. Who can do Quote mint? And when is it done?

4. Single sided rebalancing - how will it ensure delta neutrality?

5. Who is paying fees on opening and closing of perps - where is that fees being deducted from?

6. When `withdraw` or `deposit` is paused -> contract reverts -> withdrawal stops -> this will be a challenge -> how will this be mitigated??

7.
