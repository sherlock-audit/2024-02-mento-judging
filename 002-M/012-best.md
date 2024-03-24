Overt Sangria Gorilla

medium

# A third-party may lock on behalf of another user and spam their delegations

## Summary

The contract `Locking` allows a sender to lock the funds on behalf of another user.
By doing so, the sender may spam the `totalSupplyLine` and the user's balances,
in exchange for dust amounts (70-100 wei) + the gas fees. The sender may even
delegate to themselves to keep their voting power.

## Vulnerability Detail

A sender `Eve` can simply call `lock(Alice, Eve, dust, slope, cliff)`
of the contract `Locking`. As a result, `Alice` receives `dust` locked tokens,
and `Eve` increases her voting power.

## Impact

There are several implications:

 1. An attacker can spam the balance of another user by locking the dust amounts.
   This would potentially increase the gas fees of traversing through the lines
   and points.

 1. Perhaps, more importantly, an attacker can spam the list of delegations of a
   specific user. This user could be a decision-making voter in the governance process.
   From the UX perspective, it would be harder for the voter to redelegate their voting
   power by looking through a large list of spam delegations. Even though the victim
   could re-delegate the voting power to themselves, it probably would not make economic
   sense for them, as they would have to spend a lot in the gas fees for a tiny change
   in the voting power.

## Code Snippet

https://github.com/sherlock-audit/2024-02-mento/blob/24a0007769144859bb9e74b5126d4780047ef7c9/mento-core/contracts/governance/locking/Locking.sol#L66-L77

https://github.com/sherlock-audit/2024-02-mento/blob/24a0007769144859bb9e74b5126d4780047ef7c9/mento-core/contracts/governance/locking/LockingBase.sol#L253-L264

## Tool used

Manual review

## Recommendation

 1. Restrict `lock` to the delegating account and a list of trusted addresses such as the proxies.

 1. Increase the minimum amount to be locked. The current minimal amount is in the range of 50-70,
    see [LockingBase.getLock](https://github.com/sherlock-audit/2024-02-mento/blob/24a0007769144859bb9e74b5126d4780047ef7c9/mento-core/contracts/governance/locking/LockingBase.sol#L253-L264)