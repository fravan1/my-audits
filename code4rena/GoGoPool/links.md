## Important Links

1. [Code4rena Contest Link](https://code4rena.com/contests/2022-12-gogopool-contest)
2. [Github code](https://github.com/code-423n4/2022-12-gogopool)
3. [Audit Scope](https://multisiglabs.notion.site/C4-Audit-Scope-f26381cf715b41df809e0e18963baa03)

```
# Install a few tools we use to run the repo
brew install just
brew install jq
curl -L https://foundry.paradigm.xyz | bash
foundryup
forge install

git clone https://github.com/code-423n4/2022-12-gogopool.git
cd 2022-12-gogopool
yarn

# FYI We use [Just](https://github.com/casey/just) as a replacement for `Make`
just build
just test
forge test --gas-report
```
