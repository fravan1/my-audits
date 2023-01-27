- allows anyone to create stable coins backed by basket of ERC20 tokens
- these stable currncies are called RTokens
- RTokens can be minted by depositing entire basket of collateral backing tokens
  and redeemed for entire basket as well
- RToken will trade at market value of assets backing it - any difference can be arbitraged

- Can be overcollateralized
- overcollateralization by RSR holders who can stake RSR token against any RToken
- Staked RSR can be seized if collateral defaults (under collateralized)
- only based on oracles -> no governance

- RTokens generate revenue -> incentive for RSR stakers
- revenue -> yield from lending collateral tokens
- governance can direct portion of revenue to RSR stakers

Lets look at all files

`Component.sol`

- Basic building blocks
- contain state that must be migrated during upgrades
- delegate ownership to main owenr

`RToken.sol`

RTokenP1

- Inherits ERC20MetaUpgradeable
- Each RToken comes with a mandate (eg. capital preservation above all else)
- issuance rate - fraction of supply issued per block. must be < max rate
- 