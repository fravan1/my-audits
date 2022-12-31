## How does it work

- Papr is minted by borrowers, who transfer some collateral to a papr controller
- Borrowers can then exchange their papr for other assets using a decentralized exchange (e.g. Uniswap)
- Papr interest rates and the papr trading price are in a constant feedback loop
- Interest rates are programmatically updated on chain as a function of papr’s trading price on Uniswap
- (the lower the trading price, the higher the interest to borrowers)
- interest rates in turn affect the trading price, as borrowers open and close loans in response to rates.

- Interest accrues to the value of papr itself: over time, new borrowers are allowed less papr for the exact same collateral

- When closing a loan, borrowers repay the exact same amount of papr that they minted

- However, due to interest charges, it is expected that the market value of papr will have risen since they opened their loan.

- To the extent that borrower incentives push the trading price of papr up over time, corresponding to these interest charges, papr holders are rewarded.

- Current protocols only accept collateral which has

  - manipulation resistant trading volume
  - Liquidity to support selling large amounts of collateral near the spot price

- consequence of these limitations are extremely apparent in the NFT-backed loan space - generally lend to fewer than 10 NFT collections.

- market interest is there for a much wider nft market

- peer-to-peer models pale in comparison to the peer-to-protocol.

-
