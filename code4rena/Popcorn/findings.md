1. Re-entrancy risk in claimRewards - HIGH RISK

Here is case: - Reward token is ERC777 - Derives from ERC20 - `claimRewards` function -> say Bob has 10 rewards accrued -> Bob has an escrow of 1 year - `Reward token` in MultiRewards staking is 100 ERC777 - all funds in pool can be drained by re-entrancy

Prove this!!!!!

2. In Template struct -> an inline comment states that

```
  /// @Notice implementations can only be cloned if endorsed
  bool endorsed;
```

However, no such check is done in CloneFactory where an implementation is cloned

Either remove the comment OR add a check inside the clone()
