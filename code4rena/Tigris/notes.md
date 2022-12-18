# NOTES

## About Tigris

- Tigris is a leveraged trading platform that utilizes price data signed by oracles off-chain to provide atomic trades and real-time pair prices.
- Open positions are minted as NFTs, making them transferable.
- Governance via governance NFT holders

- Oracles aggregate real-time spot prices from CEXs and sign them
- Traders include the price data and signature in the trade txs.

- users can deposit collateral for up to 365 days and receive tigUSD tokens
- They will receive trading fees through an allocation of Governance NFTs
- distributed based on amount locked and lock period

- trades crypto, commodities and forex
- can set stop loss and TP levels
- funding fee is deducted every second - fee paid to the opposite OI side of your trade
- there are no price impacts and no slippage for trades as orders are instantly executed in only one transaction

- **The moment you click the 'Open Position' button, the asset price is locked in and you need to sign the transaction in your wallet within 10 seconds.**

- When opening a trade funding fees are shown right under the Open Short/Long Position button.

- Every block the trades with more open interest pay the trades on the opposite side proportionally to their size.

- If there is no one on the other side of your trade the funding fees will be detracted from your PnL.

```
Annual XSideOiFunding% = (XSideOi-YSideOi)*baseFundingRate%/XSideOi
If negative: XSideOiFunding% = XSideOiFunding%*(100%-VaultFunding%)
```

_Q. what is `vaultFunding` in above formula?_

- Please remember the moment you click the "Close" button to close an open position, the asset price is locked in and you need to sign the transaction in your wallet within 10 seconds.

- Stop orders can't be opened when a pair is closed.

- Stop order price is not guaranteed.

- There is no fee for opening a trade while the closing fee is 0.20% on commodities and crypto pairs and 0.04% on forex.

- You can decide to not use a Stop Loss or Take Profit orders.

- If you don't put SL trade will be liquidated if market price hits your liquidation price and it won't close if the PNL reaches 500%.

- Can edit stop loss or TP for an existing open trade - again, txn needs to be signed within 10 seconds - user will have to pay gas fees

- you can partially close a trade. again, on closing, user needs to sign within 10 seconds as the asset price will be locked

- you can add margin to an existing position

- adding margin to current posituion will lower leverage. although position size will remain same

- if you are adding to a long trade -> your liquidation price will go higher
- if you are adding to a short trade -> your liquidation price will go lower

- Likewise you can remove margin from account -> increases leverage
- you can choose stablecoin you want to receive (removing margin) or pay (adding margin)

**Liquidation**

- if you haven't setup SL, trade will be liquidated if P/L reaches -90%
- Remember - liquidation price takes into account funding costs -> in that sense, liquidation price is dynamic

- Liquidations are performed by bots that are paid with 0.01% of the position size.
- The rest remains inside the StableVault to increase its collateralization levels. (??? -> so a trader gets nothing after liquidation -> what do they mean by remaining stays in stable vault?)

- After a liquidation the NFT that represented the trade it's burned ---> how? who burns it? if its held by trader -> how does burn happen without approval??

**Oracle**

-Assets are priced using a Distributed Signature-Based Oracle Network that fetches and aggregates real-time CEX spot market prices.

- Oracle nodes are connected to the Internet Computer and they log all asset prices and other related data to a canister, where the data can be easily read and used on Tigris. (?? - what cannister??)

- Asset related data consists of

  - price
  - timestamp
  - node address (node that submits price)
  - market open/close status
  - oracle signed version of this data

- Upon placing a trade, price data and signatures are read from the canister and then included in the input parameters, where the validity of the data and the signatures are verified on-chain.

- Prices provided by the oracle network are also compared to Chainlink's public price feeds for additional security

- **If prices have more than a 2% difference the transaction is reverted.**

- _Reliable price_ : The moment you click the 'Open Trade' button, the asset price is locked in. **This means there is no slippage in price, ever**

-_Instant order execution_: Unlike traditional oracle-callback solutions, which require at least two transactions (one to place an order, the other for settlement), all orders are executed instantly in only one transaction, so there is no need to wait for settlement at all.

**TigUSD & Stablevault**

- The StableVault's purpose is to allow for many tokens of the same kind (such as USDC/DAI/USDT) to be used to trade on Tigris without splitting up the liquidity into multiple individual pools.

- Opening a trade on Tigris will automatically deposit the margin into a StableVault that supports it and closing a trade will withdraw from the StableVault if the trader chooses to receive a non-native stablecoin.

- Depending on StableVault collateralization level, trading fees are distributed between Vault and Governance NFT holders as:

<100%: 100% of fees go to StableVault;
100-114%: 50/50 split;

> 115%: 100% of fees go to Gov NFTs

- Thus, negative PnL and liquidations help collateralize the stablecoin while positive PnL does the opposite

- All StableVault liquidity is (for now) protocol-owned, making the growth of liquidity very sustainable.

- On Polygon the stablecoins are DAI and tigUSD, while on Arbitrum they are USDT and tigUSD.

**Staking**

- Users can deposit tigUSD to provide liquidity to the StableVault.

- 20 Governance NFTs from the Treasury allocation are deposited in the staking contract and their rewards are given to stakers.

- Staking is capped to 200k. Allowing for more liquidity would result in the apy being too low.

- Staked tigUSD acts as counterparty to traders. Winning traders will be paid with this liquidity too.

- Profits from trading fees are paid out to NFT holders in real-time.

- Governance NFT holders do not have to stake the NFTs to earn, they only need to be held in the wallet.

- Earned fees can be claimed through Tigris's website or by interacting with the Governance NFT contract directly. Rewards are paid out in Tigris stablecoins.

- Tigris has integrated LayerZero for bridging NFTs.

- NFT holders only earn the profits generated by the platform on the chain that the NFT is on.

- Can be bought from treasury or in open market at opensea

- NFT total supply is 10,000.

- Team and treasury NFTs will be minted on every sale to keep the ratio consistent.

- 16% Treasury
  18% Team
  66% Investors

## Developer Notes

Contract deployment order

-> StableToken - verified
-> PairsContract - verified
-> Referrals - verified
-> Position - verified
-> GovNFT - verified
-> StableVault - verified
-> Trading - verified
-> Forwarder - done
-> NFTSale
-> TimeLock
-> PrintContracts

### StableToken.sol

- Inherits from ERC20 Permit
