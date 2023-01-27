# 1. `Mint` function does not update `LiquidityPosition` state of caller before minting LP tokens.

## Risk Classifcation

Medium

---

## Lines of Code

[Pool.sol Line 302](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L302)

[LiquidityPosition.sol Line 147](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/LiquidityPosition.sol#L60)

---

## Vulnerability Details

### Impact

When a LP mints V2 Pool tokens, `mint` function in [PoolLibrary](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L302) gets called. Inside this function, `updateDurationWeightBeforeMaturity` updates global `short`, `long0` and `long1` fee growth.

Change in global fee growth necessitates an update to `LiquidityPosition` state of caller (specifically updating fees & fee growth rates) when there are state changes made to that position (in this case, increasing liquidity). This principle is followed in functions such as `burn`, `transferLiquidity`, `transferFees`. However when calling `mint`, this update is missing. As a result, `growth` & `fee` levels in liquidity position of caller are inconsistent with global fee growth rates.

Inconsistent state leads to incorrect calculations of long0/long1 and short fees of LP holders which inturn can lead to loss of fees. Since this impacts actual rewards for users, I've marked it as MEDIUM risk

### Proof of Concept

Let's say, Bob has following sequence of events

- MINT at T0: Bob is a LP who mints N pool tokens at T0

- MINT at T1: Bob mints another M pool tokens at T1. At this point, had the protocol correctly updated fees before minting new pool tokens, Bob's fees & growth rate would be a function of current liquidity (N), global updated short fee growth rate at t1 (s_t1) and Bob's previous growth rate at t_0 (b_t0)

- BURN at T2: Bob burns N + M tokens at T2. At this point, Bob's fees should be a function of previous liquidity (N+M), global short fee growth rate (s_t2) and Bob's previous growth rate at t_1(b_t1) -> since this update never happened, Bob's previous growth rate is wrongly referenced b_t0 instead of b_t1.

Bob could collect a lower fees because of this state inconsistency

---

## Tools used

Manual

---

## Recommended Mitigation Steps

Update the liquidity position state right before minting.

After [line 302 of Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L302), update the LiquidityPosition by adding

```
liquidityPosition.update(pool.long0FeeGrowth, pool.long1FeeGrowth, pool.shortFeeGrowth);
```

# 2. Loss of Long tokens when users send excess tokens to pool while minting LP tokens.

## Risk Classifcation

High

---

## Lines of Code

[Pool.sol Line 359](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L359)

[TimeswapV2Pool.sol Line 293](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L293)

[Error.sol Line 252](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-library/src/Error.sol#L152)

---

## Vulnerability Details

### Impact

Protocol currently uses 2 levels of callbacks for minting:

- Inner Level callback: `timeswapV2PoolMintChoiceCallback` function in [Line 349 of Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L349) that allows users to choose `Long0` and `Long1` amounts such that `long0 + long1(converted based on strike) <= long`. This is enforced by `checkEnough` function in Line359

Purpose of this callback is to allow users the freedom to choose composition of long0 & long 1 amount (since AMM only cares about the sum & not individual amounts)

- Outer level callback: `timeswapV2PoolMintCallback` function in [Line 277 of TimeswapV2Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L277) expects user to send to the pool respective amounts of long0, long1 (determined by previous callback) and short tokens corresponding to the liquidity position minted.

Purpose of this callback is to ensure user sends right amount of long0/long1/short tokens. `checkEnough` functions in Line 293/295 and 297 ensure this.

Problem with this 2 level design is that users can choose `long0` and `long1` tokens such that `long0 + long1 > long` in first callback.

In such as case, pool forces uses to send this higher sum of `long0 + long1` tokens in second callback and yet mints LP tokens only corresponding to the lower sum (original `long` amount).

Excess tokens sent to the pool are not refunded back to user. Current implementation mints a fixed `liquidity` irrespective of actual number of long0/long1 tokens sent. Since this can directly result in loss of tokens, I've categorized this issue as `HIGH` risk

### Proof of Concept

- Bob uses `mint` function using `GivenLiquidity` setting where `delta` corresponds to the amount of liquidity (lets say delta = 100)

- To mint `100` tokens, Constant Product AMM expects say 50 Long tokens and 2 short tokens
- In the first callback, user sends 55 Long0 tokens & 0 Long1 tokens. Since 55 (Long0 + Long1) > 50 (Long) tokens, callback is successful & `pool.mint` function returns long0 = 55, long1 = 0, short = 2, liquidity =100

- Now the second callback, user is expected to send 55 Long0, 0 Long1 & 2 short tokens to the pool & in exchange get 100 LP tokens

- Note that the 5 additional Long0 tokens are never refunded back to the user & permanently lost

---

## Tools used

Manual

---

## Recommended Mitigation Steps

`Long0` & `Long1` amounts returned by inner callback inside the `mint` function of [PoolLibrary](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L359) should be checked for excess.

If there is excess, `Long0` & `Long1` should be proportionately scaled down before passing these variables to second callback.Following code snippet can be considered after Line 359 in `Pool.sol`

```
        Error.checkEnough(StrikeConversion.combine(long0Amount, long1Amount, param.strike, false), longAmount);

        //****** ADD THIS *******
        uint256 longSum =  StrikeConversion.combine(long0Amount, long1Amount, param.strike, false);
        uint256 excess = longSum - longAmount;
        uint256 scalingFactor;

        if(excess > 0){
            scalingFactor =  FullMath.mulDiv(excess, uint256(1) << 16, longSum);
            long0Amount = FullMath.mulDiv(long0Amount, (uint256(1) << 16).unsafeSub(scalingFactor), uint256(1) << 16 );
            long1Amount = FullMath.mulDiv(long1Amount, (uint256(1) << 16).unsafeSub(scalingFactor), uint256(1) << 16 );
        }

        //***********************

        if (long0Amount != 0) pool.long0Balance += long0Amount;
        if (long1Amount != 0) pool.long1Balance += long1Amount;

        liquidityPosition.mint(liquidityAmount);
        pool.liquidity += liquidityAmount;

```

# 3. User receieves lesser number of Long Tokens on burning Pool liquidity resulting in loss of user funds

## Risk Classifcation

High

---

## Lines of Code

[TimeswapV2Pool.sol Line 335](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L335)

[Pool.sol Line 438](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L438)

[Error.sol Line 252](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-library/src/Error.sol#L152)

---

## Vulnerability Details

### Impact

Protocol currently uses 2 levels of callbacks for burning Pool liquidity:

- Inner callback - `timeswapV2PoolBurnChoiceCallback` function in [Line 438 of Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L438) allows user to specify `long0` & `long1` amount such that `long0 + long1 < longAmount`. `longAmount` here refers to the amount of long tokens calculated by Constant Product AMM to be returned to user for burning a specifc amount of pool liquidity. This check is done by `checkEnough` function in line 450.

Purpose of this callback is to provide users a choice to decide composition of long0 & long1 (protocol only cares about the sum of tokens & not individual levels).

- Outer callback - the `long0`, `long1`, `short` tokens returned by `pool.mint` function are then passed to second callback in [Line 343 of TimeswapV2Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L343). Note that pool also transfers `long0` and `long1` tokens to the user.

Problem here is that if user passes a lower number of `long0` and `long1` tokens such that `long0 + long1 < long`, proportion of long tokens returned to the user will be inconsistent with the liquidity burnt by the pool. While the liquidity burnt is consistent with the constant AMM calculation corresponding to `long` amount, actual tokens sent by the pool are the numbers chosen by user in inner callback.

This inconsistent leads to a direct loss of tokens for users when they choose lower values of `long0` & `long1` tokens. Due to a potential loss of user tokens, I've marked this as `HIGH` risk

### Proof of Concept

- Bob requests a burn of 100 LP tokens. Constant Product AMM shows pool needs to send 50 Long tokens & 2 Short tokens back to user
- Bob chooses 40 Long1 tokens, 0 Long2 tokens -> this satisfies the callback sanity check (long1 + long0 (40) < long(50))
- Pool then expects Bob to transfer 100 LP tokens for burning
- Pool returns Bob 40 Long0, 0 Long1 & 2 Short tokens

Notice that while 100 LP tokens are burnt, only 40 Long0 tokens are received back. 10 Long0 tokens belonging to user are never received.

---

## Tools used

Manual

## Recommended Mitigation Steps

Liquidity calculated in the `Pool.burn` function should be scaled down proportionately if lesser sum of `long0 + long1` tokens are sent by the user. Just like liquidity, `shortAmount` should be adjusted downwards proportionate to the shortfall

This will ensure user does not lose LP tokens without receiving proportionate long tokens in return

Recommend the following code block after line 540 in `Pool.sol`

```
        Error.checkEnough(longAmount, StrikeConversion.combine(long0Amount, long1Amount, param.strike, true));
        //************ ADD THIS CODE BLOCK **************
        uint256 longSum =  StrikeConversion.combine(long0Amount, long1Amount, param.strike, false);
        uint256 shortFall = longAmount - longSum;
        uint256 scalingFactor;

        if(excess > 0){
            scalingFactor =  FullMath.mulDiv(shortFall, uint256(1) << 16, longAmount);
            liquidityAmount = FullMath.mulDiv(liquidityAmount, (uint256(1) << 16).unsafeSub(scalingFactor), uint256(1) << 16 );
            shortAmount = FullMath.mulDiv(shortAmount, (uint256(1) << 16).unsafeSub(scalingFactor), uint256(1) << 16 );
        }


        //***********************************************

        if (long0Amount != 0) pool.long0Balance -= long0Amount;
        if (long1Amount != 0) pool.long1Balance -= long1Amount;

        pool.liquidity -= liquidityAmount;
```

# 4. Malicious borrower can create pool imbalance & lower rates by tricking the V2 pool to send low number of long tokens in exchange of short tokens

## Risk Classifcation

High

---

## Lines of Code

[TimeswapV2Pool.sol Line 453](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L453)

[Pool.sol Line 615](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L615)

---

## Vulnerability Details

### Impact

Timeswap V2 Pool works on constant product AMM where the total long tokens & short tokens follow the equation `total long * total short = L`. Any increase in long tokens has to be accompanied with a proportionate drop in short tokens (and viceversa) to ensure that pool functions normally.

`Leverage` function can be exploited by a borrower to imbalance the pool by tricking the pool into sending a very small number of long tokens in exchange for short tokens. In effect, tokens can be exchanged in any ratio irrespective of the constant product AMM constraint creating an imbalance.

While this will surely cause a loss to borrower (trapped collateral can never be redeemed because of loss of long tokens), such an imbalance can distort marginal interest rates and cause losses to LPs and lenders

`leverage` function in [TimeswapV2Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L430) uses 2 levels of callbacks that control how many `short` tokens are deposited into the pool and how many `long0` and `long1` tokens are transferred out of the pool.

- Inner callback: `timeswapV2PoolLeverageChoiceCallback` in [Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L615) allows users to specify composition of `long0` and `long1` tokens subject to `long0 + long1 < long`.

Purpose of this callback is to give user a choice of what tokens (s)he intends to withdraw subject to a cap on the sum of long amount (which is calculated from constant product AMM)

- Outer callback: `timeswapV2PoolLeverageCallback` in [TimeswapV2Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L453) expects users to deposit required amount of short tokens. Post callback, function checks if these short tokens are sent using `checkEnough` function in Line 461

A malicious borrower who willingly chooses to lose collateral can request a negligible `long0` & `long1` tokens in inner callback such that `long0 + long1 <<<long`. This would mean that while proportionate short tokens go into the pool, negligible long tokens come out of the pool. This would mean the marginal interest rate `I` which is proportion of `z` tokens to the sum of `x+y` tokens will be distorted - this malicious borrowing doesn't allow interest rate to increase as much as it should.

Net effect is, new borrowers can borrow at a lower interest rate than they should (based on AMM formula), leading to a direct loss to lenders and liquidity providers. Key vulnerability is that the protocol does not check if proportionate amount of short & long tokens are exchanged and solely relies on user choice.

### Proof of Concept

I will use white paper example

- Suppose there are 160,000 long USD and 20 Short in a ETH:USD pool (strike: 800, maturity: 1 year, $ETH spot 2000)
- Based on above, x = 0, y = 200 (160k/800), z = 20 / 31557600 seconds (1 year)
- Bob wants to borrow $16k -> delta_y = 20 (16000/800)
- d\*delta_z = 2.857 short tokens
- Bob deposits 2.857 ETH to mint 2.857 long ETH and 2.857 short
- Bob ideally should deposit 2.857 short to receive 20 delta_y which would be combined with 20 ETH & sent to V2 option pool to receive 16000 USD
- Instead of the above step, Bob sends 2.857 short & requests 0.001 delta_y (negligible). Pool gets tricked into receiving 2.857 short but transfering only 0.001 delta_y to Bob
- Bob never uses the delta_y & 20 ETH to borrow $16k (doesn't have enough delta_y to execute this txn)

- Marginal interest rate at the end of the transaction should have been 22.857/180 (z+dz/ y+dy). After pool distortion above, marginal interest rate is 22.857/200 (`dy` was negligible), lower than what they should be.

- Bob's collateral is stuck & cannot be redeemed even by protocol - marginal interest rates are lesser than what they are supposed to be

---

## Tools used

Manual

---

## Recommended Mitigation Steps

Inner callback in [Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-pool/src/structs/Pool.sol#L615) needs to check if proportionate amount of `long0` and `long1` amounts are provided by user. If user choices are inconsistent with total amount of long tokens, user choice should be overriden to protect the sanctity of the constant product AMM pool.

Recommend following code block after line 626

```
  Error.checkEnough(longAmount, StrikeConversion.combine(long0Amount, long1Amount, param.strike, true));

  //*****RECOMMEND FOLLOWING***//
  uint256 longSum = StrikeConversion.combine(long0Amount, long1Amount, param.strike, true);
  uint256 shortFall = longAmount - longSum;

  if(shortFall > 0){
    long0Amount = FullMath.mulDiv(long0Amount, longAmount, longSum);
    long1Amount = FullMath.mulDiv(long1Amount, longAmount, longSum);
  }

  //********************//
```

Essentially, we are scaling up long0Amount/long1Amount to be consistent with amount of shorts exchanged.

# 5. Malicious lender can create pool imbalance by tricking V2 pool to accept disproportionately large number of long tokens in exchange for short tokens

## Risk Classifcation

High

---

## Lines of Code

[TimeswapV2Pool.sol Line 403](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L403)

[Pool.sol Line 532](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L532)

---

## Vulnerability Details

### Impact

Timeswap V2 Pool works on constant product AMM where the total long tokens & short tokens follow the equation `total long * total short = L`. Any increase in short tokens caused by lenders has to be accompanied with a proportionate drop in long tokens to keep the interest rates/pool balanced.

`Deleverage` function can be exploited by a lender by tricking the pool into accepting a very large number of long tokens in exchange for short tokens. In effect, lender exchanges a large amount of long tokens in any ratio irrespective of the constant product AMM constraint, thus creating an imbalance.

While this will surely cause a loss to lender (lending amount cannot be fully redeemed because of loss of long tokens), such an imbalance can distort marginal interest rates and cause losses to LPs and lenders

`Deleverage` function in [TimeswapV2Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L403) uses 2 levels of callbacks that control how many `long0`/`long` tokens are deposited into the pool and how many `short` tokens are transferred out of the pool.

- Inner callback: `timeswapV2PoolDeleverageChoiceCallback` in [Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/structs/Pool.sol#L532) allows users to specify composition of `long0` and `long1` tokens subject to `long0 + long1 >= long`.

Purpose of this callback is to give user a choice of what tokens (s)he intends to deposit subject to a floor on the sum of long amount (which is calculated from constant product AMM)

- Outer callback: `timeswapV2PoolDeleverageCallback` in [TimeswapV2Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L403) expects users to deposit required amount of long tokens (as defined in choice callback). Post callback, function checks if these long tokens are sent using `checkEnough` function in Line 411 and 413

A malicious lender who willingly chooses to lose the loan amount can deposit a large number of `long0` & `long1` tokens in inner callback such that `long0 + long1 >>>long`. While proportionate short tokens leave the pool, a disproportionately large number of long tokens enter the pool.

This would mean the marginal interest rate `I` which is proportion of `z` tokens to the sum of `x+y` tokens will be distorted - marginal interest rate would fall by much more than it should as per AMM (`z` decreases, but `x+y` increases by much more)

### Proof of Concept

I use the whitepaper example here:

- Suppose there are 160,000 long USD and 20 Short in a ETH:USD pool (strike: 800, maturity: 1 year, $ETH spot 2000)
- Based on above, x = 0, y = 200 (160k/800), z = 20 / 31557600 seconds (1 year)
- Bob deposits 20,000 USD into Timeswap option pool to get 20,000 Long USD and 25 shorts (20000/800)
- Bob wants to lend 10,000 USD -> (delta_y = 12.5, duration \* delta_z = 1.1764)
- Instead of sending 12.5 delta_y (10000 long USD), Bob sends 25 delta_y (20000 long USD)
- Inexchange, pool sends Bob 1.1764 short. Bob's net short is 26.1764 shorts
- Pool z = (20-1.1764) / 31557600, y = 225 (not that this should have been 212.5 ideally). Pool was tricked into accepting 12.5 additional y
- Marginal interest rate should have been (20-1.1764)/ 212.5. Instead it is now (20-1.1764)/225. This leads to lower rate for borrowers
- Net result - Bob's lending capital is stuck in the pool & cannot be redeemed. Pool balance is distorted leading to lower than usual rates for new borrowers

---

## Tools used

Manual

## Recommended Mitigation Steps

`Pool.leverage` should restrict the ability of lenders to deposit `long0` and `long1` tokens that are inconsistent with the `short` amount exchanged. If user choice is off by a wide margin, scale values of `long0` and `long1` back proportionately such the `long0 + long1 == longAmount`. Keeping the exchange proportional is necessary for balanced rates & self-correcting AMM pool.

Recommend following code block after line 535 in Pool.sol

```
        Error.checkEnough(StrikeConversion.combine(long0Amount, long1Amount, param.strike, false), longAmount);

        //*****RECOMMEND FOLLOWING***//
        uint256 longSum = StrikeConversion.combine(long0Amount, long1Amount, param.strike, true);
        uint256 excess = longSum - longAmount;

        if(excess > 0){
            long0Amount = FullMath.mulDiv(long0Amount, longAmount, longSum);
            long1Amount = FullMath.mulDiv(long1Amount, longAmount, longSum);
        }

        //********************//

        if (long0Amount != 0) pool.long0Balance += long0Amount;
        if (long1Amount != 0) pool.long1Balance += long1Amount;



```

# 6. A malicious rebalancer can imbalance the `constant sum` nature of long token pool by depositing a larger number of input tokens for a smaller number of output tokens. This also imbalances the `constant product` nature of 3 token pool

## Risk Classifcation

High

---

## Lines of Code

## [TimeswapV2Pool.sol Line 497](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L497)

## Vulnerability Details

### Impact

`Token0` and `Token1` follow the properties of a `constant sum` pool. delta_x (token0 change) is balanced proportionately by delta_y (token1 change adjusted for strike). Arbitrageurs can use the `constant sum` property to rebalance the pool based on the level of `strike` viz-a-viz the DEX spot

This balance can be broken when a malicious rebalances deposits a disproportionately large amount of a specific token inexchange for a smaller amount of alternate token. Doing this will increase the total number of long tokens in the pool & will further imbalance the `constant product` properties of 3-tooken pool that includes short tokens.

`rebalance` function in [TimeswapV2Pool.sol](https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-pool/src/TimeswapV2Pool.sol#L469) has a callback that expects the rebalancer to transfer `input` tokens into the pool inexchange for `output` tokens. `checkEnough` function on line 510 checks if the number of input tokens deposited by user is higher than the `target` amount (calculated to ensure the constant-sum nature of pool).

There is no cap on deposits as user can deposit any amount of `input` tokens so long as the number of deposited tokens is atleast equal to target. A malicious user can deposit a large number of `input` tokens creating an imbalance in the pool & breaking the `(x+y)*z = K` nature of 3 token pool.

User would no doubt lose his tokens but will distort the interest rates & pool balances that can cause losses to LP's and lenders of pool.

### Proof of Concept

I pick same example from white paper

- Suppose there are 100 long ETH and 100,000 long USD (x=100, y=100). strike is 1000 USD and spot is 1800 USD
- Bob mints 200,000 long USD and 200 short by depositing 200,000 USD
- Bob withdraws 100 long ETH (delta_x = 100) from pool
- Bob deposits 200,000 long USD (delta_y = 200) into the pool (there is no limit to how much (s)he can deposit into the pool so long as she deposits atleast 100,000 long USD)
- New pool now has x+y= 300 (earlier state of the pool had x + y = 200). Keeping the same `z`, Bob has increased the supply of long tokens by 50%. This causes imbalance to the constant product AMM & distorts the marginal interest rates (note that borrower rates are a function of z/(x+y))

---

## Tools used

Manual

---

## Recommended Mitigation Steps

`checkEnough` in Line 510 should instead be `checkExact` to ensure that constant sum properties of the long token pool are persistent. Restrict the ability of rebalancer to change pool dynamics by checking deposit amount post callback
