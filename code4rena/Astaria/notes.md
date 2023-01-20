- Strategists, LPs and borrowers are 3 players

**Strategists**

- Strategists propose loan terms
  - NFT
  - Amount in a specific asset
  - Term (in seconds)
  - rate (in bps, simple interest only)
  - Max potential debt (principal + interest at time of origination)
  - Liquidation initial ask
- Private vaults - strategists provide their own capital
- Public vaults - only whitelisted strategists can open - LPs can deposit capital
  (accept funds )
- To create a vault, strategist creates a term sheet (strategy)
- loans are repaid by order of seniority upon liquidation

**Epochs**

- epochs relevant to public vaults
- operate on a time based epoch system
- epoch length defined by strategist
- duration of new loan CANNOt exceed end of next epoch (max duration = 2 epochs)
- also used to restrict withdrawals of LPs
- LPs must signal that they wish to remove their capital at least **one epoch in advance**
- when they wish to remove capital, lps burn `VaultTokens` and receive `WithdrawTokens`

- lps continue to earn interest until next epoch begins (even after conversion to withdraw tokens)
- At beginning of next epoch, `PublicVault` computes value of their burned shares
- New loan originations are then frozen until the `PublicVault` has recovered enough capital to repay all withdrawing liquidity providers.

- If any liquidation auctions are occuring during an epoch boundary, recovered funds are split proportionally between the PublicVault and withdrawing liquidity providers, according to the value of their shares at the epoch boundary.

**Liquidations**

- When

  - loan length > duration
  - loan has positive debt balance

- Any actor can trigger liquidation on underlying collateral for liquidation fee
- Once sent to auction, all Lien tokens are frozen
- borrowe can still claim NFT by paying reserve amount of Lien tokens (what is this) before bid reaches this price
- Auction duration 72 hours
- Dutch auction starts at liquidatorInitialAsk set by strategist

**Refinancing**

- when borrower takes a loan, terms are locked for duration of loan
- vault strategies can be regularly updated by strategist
- if vault terms for same asset are better than what a user has borrowed at, borrower can refinance loan in a different vault
- Better terms are either one of

  - loan interest < by atleast 0.05%
  - loan duration increases by more than 14 days

- Refinancing is permissionless and can be called by borrower or strategist (???)

- Penalty is levied based on loan duration and amount of loan outstanding
- penalty = 10% of foregone interest earnings by original vault

**Flash actions**

- allows the user to unlock their underlying collateral and perform any action with the NFT as long as it is returned within the same block.

- dont have to worry about collateralization of loans
- eg UniV3 NFT holders can collect fees on their pool while still locking up their asset in the vault

**Collateral Tokens**

- represent locked NFTs in vault
- When all `LienTokens` against a `CollateralToken` are repaid, the holder may burn it to retrieve the underlying NFT.

**Lien Tokens**

- `LienTokens` are tokenized debt positions created and held by Vaults upon loan origination.
- Each LienToken contains relevant information for the loan it represents
- When a lien is fully paid off, its LienToken is burned.
- LienTokens may be purchased for a reserve amount that is equal to the remaining debt on the lien with an additional penalty to compensate the gas costs of loan origination.
- LienTokens have a payee field that is initially set to the LienToken owner (its Vault).
- The payee receives all loan payments, as well as auction funds if the underlying collateral is liquidated.
- If auction funds need to be paid out to withdrawing liquidity providers for a PublicVault, payee is permanently set to the address of a WithdrawProxy for the current epoch.

**Liquidations**

- Liquidations follow a waterfall structure for paying Lien token holders
- Lien token holders are paid off in order of seniority
- First lien token needs to be fully paid off before payment to second Lien token etc..
  Any excess funds after all LienTokens have been paid back are returned to the initial liquidated borrower.

**Vault Tokens**

- Any user can become LP and supply capital to public vault
- Get vault tokens in exchange - represent share in the vault
- yield bearing ERC 4626 VaultTokens
- VaultTokens are priced according to the implied value of the PublicVaults
- Price tracks the value as if borrowers instantly repaid principal on loan origination and linearly repaid interest for the duration of the loan.
- However, payments ahead of schedule (decreasing total interest), as well as liquidations, may discount the implied value of the PublicVaults and decrease the value of each VaultToken.

**WithdrawProxy**

- Liquidity providers must signal that they wish to withdraw funds at least one epoch in advance.

- When the first liquidity provider signals a withdraw for the next epoch, a WithdrawProxy is deployed for that epoch.

- The liquidity provider then burns their VaultTokens with the WithdrawProxy and receives an equal amount of WithdrawTokens for that WithdrawProxy.

- Any other withdrawing liquidity providers for the next epoch may burn their VaultTokens with that same WithdrawProxy to receive WithdrawTokens.

- At the next epoch boundary, the PublicVault records the value of the burned VaultTokens for each withdrawing liquidity provider in withdrawReserve

- If there are any active liquidations at the epoch boundary, WithdrawProxy will have been set as the liquidated liens' payee to collect auction funds.

- In this scenario, the PublicVault would record the ratio between withdrawing and remaining funds as the `liquidationWithdrawRatio`.

- The PublicVault pauses all loan originations until the withdrawReserve is met, either from loan repayments or recovered auction funds.

- The PublicVault then begins sending funds to the WithdrawProxy for that epoch.

- WithdrawToken holders can burn their tokens in exchange for the funds held by the WithdrawProxy.

- Since loans in the previous epoch had been capped to end before the end of the next epoch, the withdrawReserve should be met if no liquidations are triggered near the end of the epoch

- If a loan is liquidated and the auction end is scheduled after the epoch boundary when the WithdrawProxy begins accumulating funds for LPs to withdraw, then the payee for the liquidated LienToken is permanently set to the address of the WithdrawProxy to collect all auction funds.

- If any other LienTokens are liquidated before the end of the current epoch, the payee for those LienTokens are also set to the WithdrawProxy.

- The reserve value for all auctioned liens is tracked by the WithdrawProxy.

- Once all auctioned funds are recovered, the funds collected from auctions into the WithdrawProxy are proportionally split between withdrawing and remaining LPs (the WithdrawProxy and the PublicVault).

- This withdraw ratio is determined at the epoch boundary, when the funds owed to all withdrawing LPs is calculated.

**Merkle Trees**

- Each strategy will have a StrategyDetails leaf in index 0 to provide configuration data.
- Any format with StrategyDetails outside index 0 will be considered invalid

1. _Strategy Details_

StrategyDetails has following format

- `type` - should be 0 for `StrategyDetails`
- `version` - version of strategy format - 0
- `strategist` - wallet address of strategist opening vault
- `delegate` - EOA that can sign new strategy roots after Vault is initiated. Role cannot be reassigned without a new transaction. New strategies can be signed without needing a transaction.
- `public` - public or private vault
- `nonce` - starts at 0 at vault opening - incrementing nonce on chain invalidates all lower strategies
- `vault`- if vault address is `0x000` first merkle tree opening vault

```
{
    "uint8": "type",
    "uint8": "version",
    "address": "strategist",
    "address": "delegate",
    "boolean": "public",
    "uint256": "expiration",
    "uint256": "nonce",
    "address": "vault"
}
```

Example

```
{
    type: 0,
    version: 0,
    strategist: "0x8f99B0b48b23908Da9f727B5083052d5099e6aea",
    delegate: "0xf942dba4159cb61f8ad88ca4a83f5204e8f4a6bd",
    public: true,
    expiration: 1665780795,
    nonce: 0,
    vault: "0x0000000000000000000000000000000000000000"
}
```

Leaf hash for `StrategyDetails`

```
solidityKeccak256([ "uint8","uint8","address","address", "boolean","uint256","uint256", "address"],
[ StrategyDetails.type, StrategyDetails.version, StrategyDetails.strategist, StrategyDetails.delegate, StrategyDetails.public, StrategyDetails.expiration, StrategyDetails.nonce, StrategyDetails.vault ]);
```

2. _Lien Details_

- `Lien` is a subtype, so the value will not be preceded with a type value as it can only be nested in `Collateral` or `Collection` leaves.

- `maxPotentialDebt` is total debt senior to proposed loan that includes principal plus the earned interest over remaining life of the loans.

Wnen this number is 0, it means its the most senior debt out there

- `liquidationInitialAsk` - The initial ask when a piece is liquidated, which must be greater than the loan amount and potential debt for existing liens plus the new proposed lien.

```
{
    "uint256": "amount",
    "uint256": "rate",
    "uint256": "duration",
    "uint256": "maxPotentialDebt"
    "uint256": "liquidationInitialAsk"
}
```

2. _Collateral Details_

- `type` - Collateral, type = 1
- `token` - ERC721 collection
- `tokenId` - token Id of ERC721
- `borrower` - address of the borrower that can commit to the lien. If the value is `address(0)` then any borrower can commit to the lien
- `LienDetails` - lien

```
{
    type: 1,
    token: "0xef1a89cbfabe59397ffda11fc5df293e9bc5db90",
    tokenId: 4524,
    borrower: "0x8f99B0b48b23908Da9f727B5083052d5099e6aea",
    lien: {
        amount: 69420000000000000000,
        rate: 0,
        duration: 1663545600,
        maxPotentialDebt: 0
    }
}
```

3. _Collection_

- `type` - type 2 for collection
- `token` - token address
- `borrower` - address of borrower who can commit to lien. If address(0), any borrower can commit to lien

```
{
    "uint8": "type",
    "address": "token",
    "address": "borrower",
    "LienDetails": "lien"
}
```

Example

```
{
    type: 2,
    token: "0xef1a89cbfabe59397ffda11fc5df293e9bc5db90",
    borrower: "0x8f99B0b48b23908Da9f727B5083052d5099e6aea",
    lien: {
        amount: 69420000000000000000,
        rate: 0,
        duration: 1663545600,
        maxPotentialDebt: 0
    }
}
```

### Logic Flow - Tests

1. Initialize `Deploy` contract

`deploy()` function does following

- Deploy `WETH` if not deployed
- Get `SEAPORT` contract
- deploy `MultiRoleAuthority` (`MRA`) with auth = address(this)
- deploy `TransferProxy`(`TRANSFER_PROXY`) by passing `MultiRoleAuthority` above
- deploy `ProxyAdmin` (`PROXY_ADMIN`)
- deploy `LienToken` implementation (`LI_IMPL`)
- deploy `LienToken` proxy with implementation address of above implementation
  and proxy_admin address, and data that includes `initialize` function with `MRA` and `TransferProxy` above (`LIEN_TOKEN`)
- Create instance of ClearingHouse (`CLEARING_HOUSEMPL`)
- Create instance of CollateralToken (`CT_IMPL`)
- Create CollateralTokenProxy (using `CT_IMPL`, `PROXY_ADMIN` and initialize usinhg `MRA`, `TRANSFER_PROXY`
- Create instance of Private Vault (`SOLO_IMPLEMENTATION`)
- Create instance of Public Vault (`PUBLIC_VAULT_IMPLEMENTATION`)
- Create withdraw proxy instance (`WITHDRAW_PROXY`)
- Create Beacon proxy instance (`BEACON_PROXY`)
- Create instance of Astaria Router Implementyation (`AR_IMPL`)
- Create instance of astaria router proxy (`ASTARIA_ROUTER`)
- Create a array of `IcollateralToken.File` instance `ctfiles` with size 1
- initialize first element of `ctffiles` with
  `what` - `ICollateralToken.FileType.AstariaRouter`
  `data` - `abi.encode(address(ASTARIA_ROUTER))`
- Call `COLLATERAL_TOKEN.fileBatch(ctfiles)`
- Call `_setupRolesAndCapabilities()`
  `ASTARIA_ROUTER` role -> `LienToken.createLien`
  `ASTARIA_ROUTER` role -> `TransferProxy.tokenTransferFrom`
  `ASTARIA_ROUTER` role -> `CollateralToken.auctionVault`
  `ASTARIA_ROUTER` role -> `LienToken.stopLiens`
  `LIEN_TOKEN` role -> `CollateralToken.settleAuction`
  `LIEN_TOKEN` role -> `TransferProxy.tokenTransferFrom`

- Set `roles`

`ASTARIA_ROUTER` role -> `ASTARIA_ROUTER`
`WRAPPER` role -> `COLLATERAL_TOKEN`
`LIEN_TOKEN` role -> `LIEN_TOKEN`

- `LIEN_TOKEN.file( <CollateralTokenFileType>, abi.encode(address(COLLATERAL_TOKEN))`
- `LIEN_TOKEN.file( <AstariaRouterFileType>, abi.encode(address(ASTARIA_ROUTER))`

2. `ConsiderationTester` contract

- `_deployAndConfigureConsideration`
- create new instance of `conduitController`
- create new instance of `consideration` using `conduitController`
- create new instance of `conduit` using `this` address (ConsiderationTester)
- update channel of `conduitController` using `conduit` and `consideration`

3. Initialize `TestHelpers` contract (inherits `Deploy` and `ConsiderationTester`)

- Initialize `strategistOnePK`, `strategistTwoPK`, and `strategistRoguePK`
- Initialize `borrower`, `bidderPK`, `bidderTwoPK`
- Initialize `ILienToken.Details` variable `shortNSweet`

  - `maxAmount` - 150 ether
  - `rate` - 1% \* 150 / 365
  - `duration` - 1 minute
  - `maxPotentialDebt` - 0 ether
  - `liquidationInitialAsk` - 500 ether

  - Initialize `ILienToken.Details` variable `blueChipDetails`

    - `maxAmount` - 150 ether
    - `rate` - 1% \* 150 / 365
    - `duration` - 10 days
    - `maxPotentialDebt` - 0 ether
    - `liquidationInitialAsk` - 500 ether

  - Initialize `ILienToken.Details` variable `rogueBuyoutLien`

    - `maxAmount` - 50 ether
    - `rate` - 1% \* 150 / 365
    - `duration` - 10 days
    - `maxPotentialDebt` - 50 ether
    - `liquidationInitialAsk` - 500 ether

  - Initialize `ILienToken.Details` variable `standardLienDetails`

    - `maxAmount` - 50 ether
    - `rate` - 1% \* 150 / 365
    - `duration` - 10 days
    - `maxPotentialDebt` - 50 ether
    - `liquidationInitialAsk` - 500 ether

  - Initialize `ILienToken.Details` variable `refinanceLienDetails`

    - `maxAmount` - 50 ether
    - `rate` - 1% \* 150 / 365
    - `duration` - 25 days
    - `maxPotentialDebt` - 53 ether
    - `liquidationInitialAsk` - 500 ether

  - Initialize `ILienToken.Details` variable `refinanceLienDetails2`

    - `maxAmount` - 50 ether
    - `rate` - 1% \* 150 / 365
    - `duration` - 25 days
    - `maxPotentialDebt` - 52 ether
    - `liquidationInitialAsk` - 500 ether

  - Initialize `ILienToken.Details` variable `refinanceLienDetails3`

    - `maxAmount` - 50 ether
    - `rate` - 1% \* 150 / 365
    - `duration` - 25 days
    - `maxPotentialDebt` - 51 ether
    - `liquidationInitialAsk` - 500 ether

  - Initialize `ILienToken.Details` variable `refinanceLienDetails4`
    - `maxAmount` - 50 ether
    - `rate` - 1% \* 150 / 365
    - `duration` - 25 days
    - `maxPotentialDebt` - 55 ether
    - `liquidationInitialAsk` - 500 ether

- Call `setUp()`
  - test mode
  - setup `deploy` contract instance
  - setup `considerationTester` contract instance
  - `SEAPORT = ConsiderationInterface(address(consideration))`
  - run `deploy` in `deploy` to setup all above (LIEN, COLLATERAL etc)
- Label all contracts -> `WETH9`, `MRA`, `TRANSFER_PROXY`, `LIEN_TOKEN`
  `COLLATERAL_TOKEN`, `collateral conduit`, `ASTARIA_ROYTER`
- Create instance of `V3SecurityHook` -> `V3_SECURITY_HOOK` and pass
  address of `Uni V3 NFT`
  -> `V3SecurityHook` assigns `positionManager` as `nftManager`
  -> `V3SecurityHook` has a function `getState` -> returns
  encoded data containinhg `nonce`, `operator` and `liquidity` of UniV3 token ID

- Create a `ctfiles` array with size 2 (`CollateralToken.File[]`)
- In `ctfiles[0]` -> assign `what` - AstariaRouter, `data` -> abi.encode(address(ASTARIA_ROUTER))
- In `ctfiles[1]` -> assign `what` - SecurityHook, `data` -> abi.encode(UNI_V3_NFT, address(V3_SECURITY_HOOK))
- call `COLLATERAL_TOKEN`.fileBatch(ctfiles)`

- Create a new instance of `UniqueValidator` (`UNIQUE_STRATEGY_VALIDATOR`)
- Create a new instance of `CollectionValidator` (`COLLECTION_STRATEGY_VALIDATOR`)
- Create a new instance of `UNI_V3Validator` (`UNIV3_LIQUIDITY_STRATEGY_VALIDATOR`)

- Create `files` array with size 3 (`IAstariaRouter.File[]`)
- `files[0]` -> `what` - StrategyValidator, `data` - abi.encode(uint8(1), address(UNIQUE_STRATEGY_VALIDATOR))
- `files[1]` -> `what` - StrategyValidator, `data` - abi.encode(uint8(2), address(COLLECTION_STRATEGY_VALIDATOR))
- `files[2]` -> `what` - StrategyValidator, `data` - abi.encode(uint8(3), address(UNIV3_LIQUIDITY_STRATEGY_VALIDATOR)

- call `ASTARIA_ROUTER`.fileBatch(ctfiles)`

- Call `_setupRolesAndCapabilities()` to set roles and role owners

4. Initialize `AstariaTest` contract (inherits `TestHelpers` )

-
