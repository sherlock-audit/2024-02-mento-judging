Ripe Juniper Bobcat

medium

# Current parameters allow a minimum lock of only 2 blocks to get voting power which can result in governance attack

## Summary

The locking contract works in epochs of 1 week in blocks, the minimum lock is currently 1 week slope and 0 week cliff. 

A user can lock 2 blocks before the end of the epoch with minimum parameters, vote then withdraw a block later when new epoch starts.

## Vulnerability Detail

The [`lock()`](https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/locking/Locking.sol#L66) function allows to lock mento tokens for veMento, the user must lock for a minimum time which is defined by `minCliffPeriod` and `minSlopePeriod`.

Currently these variables are set to 0 and 1, which result in minimum being 1 week slope. 
Additionally the `GOVERNOR_VOTING_DELAY` that defines when voting starts for a new proposal is currently set to 0.

The issue is that the lock is calculated per week and resets at a given time (wednesday evening). This means a user could lock 2 blocks before epoch reset, create a proposal a block alter ([cannot vote and propose in the same block](https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/locking/LockingVotes.sol#L28) and then withdraw right after new epoch starts.

While the user's power will be small (1/104 of locked amount), having to lock for only 2 blocks may allow him to temporarily borrow a high amount of mento, lock them for 2 blocks, vote in governance and then withdraw and reimburse the loan.

POC:

Can be copied then pasted into `locking.t.sol`:

```solidity
function test_withdraw_whenInSlope1week() public {
    uint96 amount = 1000 * 1 ether;
    mentoToken.mint(alice, amount);

    //lock 2 blocks before epoch ends
    _incrementBlock(1 * weekInBlocks - 2);

    vm.prank(alice);
    locking.lock(alice, alice, amount, 1, 0);

    //Alice now has voting power
    assertEq(locking.balanceOf(alice), 9615380000000000000); // ~100 ether / 104

    _incrementBlock(2);

    //after two blocks alice can withdraw
    vm.prank(alice);
    locking.withdraw();

    assertEq(mentoToken.balanceOf(address(locking)), 0);
    assertEq(mentoToken.balanceOf(alice), amount);
  }
```

## Impact

Medium. While the voting power of the user will be low, being able to lock for such a short period could allow different kind of governance attack. An example would be borrowing a high amount of tokens to counter the low voting power and reimburse a block later.

## Code Snippet

https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/locking/Locking.sol#L66
https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/GovernanceFactory.sol#L211
https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/locking/LockingBase.sol#L251

## Tool used

Manual Review

## Recommendation

Consider enforcing a minimum of 2 epochs for the slope to make sure users have to stake for a minimum of 1 week + 2 blocks to have voting power.

And/Or modify the `GOVERNOR_VOTING_DELAY` to not be 0 so a minimum amount of blocks are needed between creating a proposal and voting.
