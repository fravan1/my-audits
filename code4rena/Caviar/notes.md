- Order of tests

  - DeployScript -> deploys `caviar` contract
  - Create Fake Azukis/BayC etc -> deploys `nft` contracts
  - CreatePair -> creates `pair` -> generates merkleroot and creates a pair with nft, baseToken (address(0) - eth), merkleRoot -> merkleRoot is generated for a given set of tokenIds

`LPToken.sol`

- LP token which is minted and burned by the Pair contract to represent liquidity in the pool.
- what if pairSymbol is duplicate???
- has `_mint` and `_burn` functionality
- can be called only by owner - owner in this case is `Pair` contract

`Pair.sol`

- A pair of an NFT and a base token that can be used to create and trade fractionalized NFTs.
- derives from ERC20 and ERC721TokenReceiver
- has a `onERC721Received()` that gets triggered after the `transfer` of NFT as part of safeTransferFrom

- Uses `SafeTransferLib` for `address` and `ERC20`
- Contains immutable address of `nft`, `caviar`, `baseToken`, `lpToken`
- Also contains immutable bytes32 merkleroot - cannot be changed over life of pair
- Has `add`, `remove`, `buy`, `sell`, `wrap`, `unwrap`, `close` and `withdraw` functions

  - `add` - user can add liquidity to a pair of f-tokens/base token
  - `remove` - Removes liquidity from the pair
  - `buy` - Buys fractional tokens from the pair
  - `sell` - Sells fractional tokens back to the pair
  - `wrap` - Wraps NFTs to fractional tokens
  - `unwrap` - Unwraps NFTs by burning fractional tokens
  - `nftAdd` - Adds liquidity to the pair using NFTs.
  - `nftRemove` - Removes liquidity from the pair using NFTs
  - `nftBuy` - Buys NFTs from the pair using base tokens.
  - `nftSell` - Sells NFTs to the pair for base tokens.

`Caviar.sol`

- Its `Owned` contract -> msg.sender = owner
- using `SafeERC20Namer` library for `address`
- `Create` event to create a new pair -> with nft address and base token address
- we are also emitting merkleRoot for that pair (how is this generated???)
- `Destroy` event to destroy an existing pair with nft and base token
- pairs mapping `pairs[nft][baseToken][merkleRoot] -> pair`
- pairs takes in nft address, base token address and merkle root -> and maps to a pairs contract
- `create`

  - what if I pass 0 address for nft -> 0 address for baseToken represents ETH?
  - 0 address for base token represents ETH
  - what if I send a different merkleRoot but same paid -> nft/base token

- `destroy`
  - `caviar.destroy` can only be called by the pair itself
  - the pair contract tht is stored in `pairs[nft][baseToken][merkleRoot]`

`CreateFakeAzukis.sol`

- FakeAzukis -> ERC721 contract
- Create Fake Asuki script -> derives from Script (Foundry)
- In the `run` function -> mint 250 fake azukis

`CreatePair.s.sol`

- creates a pair for a given nft, baseToken and tokenIds list
- generates a merkle root for the rankings file
- ``vm.ffi` generates merkle root on the ranking File list of tokenIds
- `generateMerkleRoot` and `generateMerkleProof` are functions used to create merkle root and proof array (2d array) for a given json file with token ids
