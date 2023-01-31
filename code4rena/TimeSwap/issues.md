1. `OptionsLibrary.burn`

   - not burning totalShorts
   - not burning long0 and long1 of msg.sender
   - not burning shorts of msg.sender

2. Where are we checking that person calling `burn` indeed has tokens to burn

3. `OptionsLibrary.swap`
   - Why isn't `option.long0` not being reduced when `option.long1` being added
   - Why isn't `option.long1` not being reduced when `option.long0` being added

Looks like Wrong accounting

3. Where are we checking that person calling `Swap` indeed has the necessary token0 and token1??? -DAMN

4. `OptionsLibrary.collect`

   - what about reducing `option.long0[msg.sender]` and `option.long1[msg.sender]` - dont matter because they only matter when time < maturity

   - what about `shorts[msg.sender]`- being adjusted correctly

5. `TimeswapV2Option.burn`

   - potential reentrancy attack -> after the transfers are made,
     we are reducing balances of option.long0[msg.sender], option.long1[msg.sender] and option.short[msg.sender]

   - run a test case to test this thoroughly

6. `TimeswapV2Option.collect`

- scope for reentrancy attack -> check if this is possible
- if i create a fallback function that again sends shortAmount

7. `ERC1155Enumerbale._removeTokenEnumeration`

   - `removeTokenFromAllTokensEnumeration` will never get executed
   - `_idTotalSupply[id] ==0`, next statement will revert ->
   - should be after the supply is changed

8. `TimeSwapV2Token.mint` allows anyone to create a V2token with 0 strike

9. `Pool.collectProtocolFees` is an external function that changes pool position

- anyone can access this funcytion and set the protocol fee parameters to 0
- after this, when pool owners can never withdraw
  HIGH

10. `Pool.transferFees` - I can block any updates to fee growth if I am a mischeivous LP -> I can call `collectTranscationFees` and pass a blockTimeStamp > maturity -> this triggers the `updateDurationWeightAfterMaturity` -> which makes the pool.lastTimestamp = maturity. After this, no changes to the pool fee growth will be updated to the global pool...
    HIGH

DONE

11. `Pool.mint` - Before minting, update of liquidityPosition for minter is missing -> this results in incorrect calculations of subsequent pool balances
    for long0 and long1 token

12. `Pool.mint` -> if a LP sends long0Amount, long1Amount such that
    long0Amount + long1Amount > longAmount, LP losses the excess Long tokens
    sent to the pool -> liquidity is calculated assuming `longAmount` of tokens
    are sent to the pool -> excess liquidity that is not backed by corresponding
    LP token should be returned back to the LP. Not doing so creates accounting
    imbalances that make the pool lose its constant product properties & causes loss of funds and IL to its LPs

13. `Pool.burn` -> if a LP sends a very minimal value of long0Amount, long1Amount -> we are burning full LP token equivalent to the delta of token
    -> and yet nothing is transferred to user -> very littl amount of Long1 tokens
    amd Long0 tokens are transferred ->

- causes loss to the LPs
- disrupts the constant AMM function (shortAmount corresponding to LP delta is burned by long amounts are not proportionate)

14. `Pool.deleverage` -> LP can send a large number of Long tokens in exchange of very few short tokens -> imbalance the pool and disrupt the rate calculation

15. `Pool.deleverage` - after pool short fee growth is updated, shouldnt we update the Liquidity position- unclear to me

16. `Pool.rebalance` - large transfer of Long0 can imbalance the pool

17. `TimeswapV2Pool.initialize` -> can anyone update the rate - what if a malicious player does this?

    - only if there is no liquidity

18. Chk what happens if I setyp a pool, option pair with 0 strike -> can i do this

19. Chk what happens if I setup a pool with protocol fees > transaction fees -> is this ok??

