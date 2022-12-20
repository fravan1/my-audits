Findings for further investigation

- what if I pass 0 address for nft in `create` function of `Caviar.sol`
- what if I send a different merkleRoot but same paid -> nft/base token -> LP token seems to be the same - wont this create a bunch of LP tokens, all with same name ??

- will `nft.tokenSymbol()` crash if nft is a zero address or a non nft??
- what if pairSymbol is duplicate in LPToken minting??
- `removeQuote` has division by 0 when lpTokenSupply = 0
- `addQuote` cannot work on a empty pool with no token reserves
- `nftRemove` uses `unwrap` with a set of tokenIds - what if token Ids while removing are different from tokenIds while adding? Can this be exploited???
