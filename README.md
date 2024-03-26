# Issue M-1: User Can Vote Even When They Have 0 Locked Mento (Edge Case) 

Source: https://github.com/sherlock-audit/2024-02-mento-judging/issues/14 

## Found by 
ComposableSecurity, HHK, Kose, ParthMandale, sakshamguruji, slylandro, y4y
## Summary

There exists an edge case where the user will be withdrawing his entire locked MENTO amount and even then will be able to vote , this is depicted by a PoC to make things clearer.

## Vulnerability Detail

The flow to receiving voting power can be understood in simple terms as follows ->

Users locks his MENTO and chooses a delegate-> received veMENTO which gives them(delegatee) voting power (there's cliff and slope at play too)

The veMENTO is not a standard ERC20 , it is depicted through "lines" , voting power declines ( ie. slope period) with time
and with time you can withdraw more of your MENTO.

The edge case where the user will be withdrawing his entire locked MENTO amount and even then will be able
to vote is as follows ->

1.) User has locked his MENTO balance in the Locking.sol

2.) The owner of the contract "stops" the contract for some emergency reason.

3.) In this stopped state the user calls withdraw() which calls getAvailableForWithdraw() here https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/locking/Locking.sol#L97

4.) Since the contract is stopped , the `getAvailableForWithdraw` will return the entire locked amount of the user as withdrawable 

```solidity
function getAvailableForWithdraw(address account) public view returns (uint96) {
    uint96 value = accounts[account].amount;
    if (!stopped) {
      uint32 currentBlock = getBlockNumber();
      uint32 time = roundTimestamp(currentBlock);
      uint96 bias = accounts[account].locked.actualValue(time, currentBlock);
      value = value - (bias);
    }
    return value;
 ```
 
 5.) The user receives his entire locked amount in L101.
 
 6.) The owner "start()" the contract again 
 
 7.) Since the user's veMENTO power was not effected by the above flow , there still exists veMENTO a.k.a voting power to the delegate,
 and the user's delegate is still able to vote on proposals (even when the user has withdrew everything).
 
 POC
 
 Import console log first in the file , paste this test in the `GovernanceIntegration.t.sol`
 
 ```solidity
 function test_Poc_Stop() public {

    vm.prank(governanceTimelockAddress);
    mentoToken.transfer(alice, 10_000e18);

    vm.prank(governanceTimelockAddress);
    mentoToken.transfer(bob, 10_000e18);

    vm.prank(alice);
    locking.lock(alice, alice, 10_000e18, 1, 103);

    vm.prank(bob);
    locking.lock(bob, bob, 1500e18, 1, 103);

    vm.timeTravel(BLOCKS_DAY);

    uint256 newVotingDelay = BLOCKS_DAY;
    uint256 newVotingPeriod = 2 * BLOCKS_WEEK;
    uint256 newThreshold = 5000e18;
    uint256 newQuorum = 10; //10%
    uint256 newMinDelay = 3 days;
    uint32 newMinCliff = 6;
    uint32 newMinSlope = 12;

    vm.prank(alice);
    (
      uint256 proposalId,
      address[] memory targets,
      uint256[] memory values,
      bytes[] memory calldatas,
      string memory description
    ) = Proposals._proposeChangeSettings(
        mentoGovernor,
        governanceTimelock,
        locking,
        newVotingDelay,
        newVotingPeriod,
        newThreshold,
        newQuorum,
        newMinDelay,
        newMinCliff,
        newMinSlope
      );

    // ~10 mins
    vm.timeTravel(120);

    

    vm.startPrank(governanceTimelockAddress);
    locking.stop();
    vm.stopPrank();

    uint bal2 = mentoToken.balanceOf(alice);
    console.log(bal2);

    vm.startPrank(alice);
    locking.withdraw();
    vm.stopPrank();

    vm.startPrank(governanceTimelockAddress);
    locking.start();
    vm.stopPrank();

    uint bal = mentoToken.balanceOf(alice);
    console.log(bal);
    vm.prank(alice);
    

    console.log(mentoGovernor.castVote(proposalId, 1));
  }
  ```
 
 You can see the Alice withdrew her entire locked amount and still was able to caste her vote.


## Impact

 User still able to vote even when the entire locked amount is withdrawn.

## Code Snippet

 https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/locking/Locking.sol#L111-L119

## Tool used

Foundry

## Recommendation

 When the entire amount is withdrawn adjust the logic to remove the corresponding lines for the delegator.



## Discussion

**nevillehuang**

Valid medium, users retain vote in an edge case scenario then holds when owners stops (pause) contract

# Issue M-2: Discrepancy Between Initialization and Setter Function for "Internal Time" Resulting in Time Management Dysfunction 

Source: https://github.com/sherlock-audit/2024-02-mento-judging/issues/19 

## Found by 
Kose
## Summary

## Vulnerability Detail
The startingPointWeek variable is responsible for maintaining internal timing consistency within the locking contract. This variable is initialized as follows in the GovernanceFactory:
```solidity
    uint32 startingPointWeek = uint32(Locking(lockingImpl).getWeek() - 1);
    TransparentUpgradeableProxy lockingProxy = ProxyDeployerLib.deployProxy( // NONCE:7
      address(lockingImpl),
      address(proxyAdmin),
      abi.encodeWithSelector(
        lockingImpl.__Locking_init.selector,
        address(mentoToken), /// @param _token The token to be locked in exchange for voting power in form of veTokens.
        startingPointWeek, ///   @param _startingPointWeek The locking epoch start in weeks. We start the locking contract from week 1 with min slope duration of 1
        0, ///                   @param _minCliffPeriod minimum cliff period in weeks.
        1 ///                    @param _minSlopePeriod minimum slope period in weeks.
      )
```
Now, let's examine the function call trace during the initialization of this variable:
```solidity
  /**
   * @notice Returns "current week" of the contract. The Locking contract works with a week-based time system
   * for managing locks and voting power. The current week number is calculated based on the number of weeks passed
   * since the starting point week. The starting point is set during the contract initialization.
   */
  function getWeek() external view returns (uint256) {
    return roundTimestamp(getBlockNumber());
  }
...
  /**
   * @notice Calculates the week number for a given blocknumber
   * @param ts block number
   * @return week number the block number belongs to
   */
  function roundTimestamp(uint32 ts) public view returns (uint32) { 
    if (ts < getEpochShift()) {
      return 0;
    }
    uint32 shifted = ts - (getEpochShift());
    return shifted / WEEK - uint32(startingPointWeek);
  }
...
  function getBlockNumber() internal view virtual returns (uint32) {
    return uint32(block.number);
  }
```
As we can observe, the getWeek function within LockingImpl returns the number of weeks elapsed since the gnosis block because currently startingPointWeek is 0 since it is not initialized yet.
Let's assume it corresponds to "179" (Test setup that I will use from protocol tests is using this value so I will continue with it) and continue.
The startingPointWeek within the locking contract is initialized with the value 179. Calculations involving the getWeek() function will operate correctly, as it deducts startingPointWeek when computing the system's internal week.
Now lets check the setStartingPointWeek() function:
```solidity
  /**
   * @notice Sets the starting point for the week-based time system
   * @param newStartingPointWeek new starting point
   */
  function setStartingPointWeek(uint32 newStartingPointWeek) public notStopped onlyOwner { 
    require(newStartingPointWeek < roundTimestamp(getBlockNumber()), "wrong newStartingPointWeek");
    startingPointWeek = newStartingPointWeek;

    emit SetStartingPointWeek(newStartingPointWeek);
  }
```
As we can see from the require statement, setStartingPointWeek() will only permit newStartingPointWeek to be less than roundTimestamp(getBlockNumber()). Currently, when computing weeks, it subtracts startingPointWeek (initialized with 179). Therefore, if there have been 10 weeks since the initialization, setStartingPointWeek() can only accept numbers smaller than 10. This assertion is confirmed by the following Proof of Concept (POC):
```solidity
  function test_notAllowChangeWeek() public {
    vm.timeTravel(BLOCKS_WEEK * 10);
    vm.prank(address(governanceTimelock));
    locking.setStartingPointWeek(11);
  }
```
Add the function provided above to **LockIntegration.fuzz.t.sol** and execute the test. You will notice that it reverts with "wrong newStartingPointWeek". This outcome aligns with the natspec comment observed during the deployment of Locking:
```solidity
        startingPointWeek, ///   @param _startingPointWeek The locking epoch start in weeks. We start the locking contract from week 1 with min slope duration of 1
```
So there is no problem so far, we can only change the startingPointWeek with values less than current internal week number. 
Let's change it and see what happens then (Again add it to **LockIntegration.fuzz.t.sol**) :
```solidity
  function test_ChangeWeekBrokesSystem() public {
    vm.timeTravel(BLOCKS_WEEK * 10);

    uint32 currentWeek = locking.roundTimestamp(uint32(block.number));
    console.log("Internal week before update:", currentWeek);

    vm.prank(address(governanceTimelock));
    locking.setStartingPointWeek(9);

    currentWeek = locking.roundTimestamp(uint32(block.number));
    console.log("Internal week after update:", currentWeek);
  }
```
Here is the console output:
>    [PASS] test_ChangeWeekBrokesSystem() (gas: 37271)
Logs:
  Internal week before update: 11
  Internal week after update: 181

As we can see internal week accounting is completely broken after a benign setStartingPointWeek() call.
This issue arises because during the deployment of the proxy, the calculation setting startingPointWeek utilized the implementation contract's variable, which is (un)initialized to 0. Consequently, it computed the variable with reference to the gnosis block. However, upon calling setStartingPointWeek(), it utilizes the proxy's storage (now set to 179), causing it to calculate the variable with reference to the internal system time. As a result, instead of producing the expected impact, it essentially disrupts the entire week accounting in the contract, leading to the unlocking of all locks.
## Impact
Broken time management in a contract that is entirely time-dependent, which in return all locks will be unlocked and other time accounting issues can occur.
## Code Snippet
[Initialization](https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/GovernanceFactory.sol/#L201)
[setter](https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/locking/LockingBase.sol#L320)
## Tool used

Manual Review

## Recommendation
To resolve the issue and ensure the system functions as intended, we have two potential solutions:

1. Modify the setStartingPointWeek Functionality:
If the desired functionality is starting the system with week 1: Update the requirement in the setStartingPointWeek() function to allow for values such as 180. This adjustment will enable the system to start with week 1 and continue functioning correctly.
2. Consistent Initialization Approach:
If the desired functionality is using the time in referance to gnosis + timeShift: Ensure consistency in initialization by calculating the startingPointWeek in the factory using a new function, rather than relying on the uninitialized implementation contract. By doing so, the setStartingPointWeek() function will be safe to call with values such as 10, maintaining the system's expected functionality.

The current conflict arises from the initialization method using the latter approach, while the change function adopts the former. This inconsistency results in system malfunction. Implementing either of these solutions will help mitigate the issue and restore the system's functionality.



## Discussion

**nevillehuang**

Could be invalid, as seen by this comment [here](https://discord.com/channels/812037309376495636/1218212138871558204/1220294134145744896), but leaving open for discussion

**baroooo**

The selection of the starting point week is primarily for convenience and lacks a technical distinction between commencing at week 1 or week 180. Our plan is to never update the starting point week unless there is a full lock migration because regardless of the problem mentioned in the finding, changing the week numbers with active locks would break the time management of the locks.

An example provided by the researcher suggested a hypothetical scenario where the governance updates the starting point from 178 to 9. It's difficult to envision a scenario that justifies such an action. Following such a change, the internal week would adjust from 11 to 181 which is expected.

The researcher correctly identified that updating the starting point week to certain values (e.g., week 180 in the given scenario) is not possible. Which is technically correct because of the require statement, but being able to change the starting week with active locks would still mess up with the timings of the locks even after applying the suggested fix.

Because of the reasons explained above, we don’t see this as a valid Medium finding but rather a Low severity. We appreciate the finding and we may remove the setStartingPointWeek since we don’t plan to to use it unless the unlikely case of a lock migration.

**nevillehuang**

request poc

To faciliate discussion between watson and sponsor, but I am inclined to believe this issue is low severity based on sponsor comments, given admin ultimately decides which starting week is set.

**sherlock-admin3**

PoC requested from @kosedogus

Requests remaining: **2**

**kosedogus**

Please create a Time.t.sol under *mento-core/test/governance/IntegrationTests* and paste the following:
```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.18;
// solhint-disable func-name-mixedcase, max-line-length, max-states-count

import { TestSetup } from "../TestSetup.sol";
import { Vm } from "forge-std-next/Vm.sol";
import { VmExtension } from "test/utils/VmExtension.sol";
import { Arrays } from "test/utils/Arrays.sol";

import { GovernanceFactory } from "contracts/governance/GovernanceFactory.sol";
import { MentoToken } from "contracts/governance/MentoToken.sol";
import { Locking } from "contracts/governance/locking/Locking.sol";
import { TimelockController } from "contracts/governance/TimelockController.sol";
import { console } from "forge-std-next/console.sol";


contract TimeTest is TestSetup {
  using VmExtension for Vm;

  GovernanceFactory public factory;

  MentoToken public mentoToken;
  TimelockController public governanceTimelock;
  Locking public locking;

  address public celoGovernance = makeAddr("CeloGovernance");
  address public celoCommunityFund = makeAddr("CeloCommunityFund");
  address public watchdogMultisig = makeAddr("WatchdogMultisig");
  address public mentoLabsMultisig = makeAddr("MentoLabsMultisig");
  address public fractalSigner = makeAddr("FractalSigner");

  bytes32 public merkleRoot = bytes32("MockMerkleRoot");

  function setUp() public {
    vm.roll(21871402); // (Oct-11-2023 WED 12:00:01 PM +UTC)
    vm.warp(1697025601); // (Oct-11-2023 WED 12:00:01 PM +UTC)

    vm.prank(owner);
    factory = new GovernanceFactory(celoGovernance);

    GovernanceFactory.MentoTokenAllocationParams memory allocationParams = GovernanceFactory
      .MentoTokenAllocationParams({
        airgrabAllocation: 50,
        mentoTreasuryAllocation: 100,
        additionalAllocationRecipients: Arrays.addresses(address(mentoLabsMultisig)),
        additionalAllocationAmounts: Arrays.uints(200)
      });

    vm.prank(celoGovernance);
    factory.createGovernance(watchdogMultisig, celoCommunityFund, merkleRoot, fractalSigner, allocationParams);
    mentoToken = factory.mentoToken();
    governanceTimelock = factory.governanceTimelock();
    locking = factory.locking();
  }

  function testTime() public {
    vm.timeTravel(BLOCKS_WEEK * 10);
    uint256 week = locking.roundTimestamp(uint32(block.number));
    console.log("Week After Deployment + 10 Week:", week);

    uint256 startingPointWeek = locking.startingPointWeek();
    console.log("startingPointWeek:", startingPointWeek);

    // startingPointWeek is 179 hence 180 or 178 are possible expected values to call this function with
    vm.prank(address(governanceTimelock));
    vm.expectRevert("wrong newStartingPointWeek");
    locking.setStartingPointWeek(180);

    vm.prank(address(governanceTimelock));
    vm.expectRevert("wrong newStartingPointWeek");
    locking.setStartingPointWeek(178);

    // But it can only be called by values less than currentWeek (which is 11 currently)
    vm.prank(address(governanceTimelock));
    vm.expectRevert("wrong newStartingPointWeek");
    locking.setStartingPointWeek(12);

    vm.prank(address(governanceTimelock));
    locking.setStartingPointWeek(9);

    week = locking.roundTimestamp(uint32(block.number));
    console.log("Week after setStartingPointWeek():", week);
  }
}
```
Output:
>    Ran 1 test for test/governance/IntegrationTests/Time.t.sol:TimeTest
[PASS] testTime() (gas: 60693)
Logs:
  Week After Deployment + 10 Week: 11
  startingPointWeek: 179
  Week after setStartingPointWeek(): 181
  Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.57ms


setUp function is taken from *LockingIntegration.fuzz.t.sol*

---------------------------------------------------------------------------------------------------------------------------------------------------------
> Could be invalid, as seen by this comment [here](https://discord.com/channels/812037309376495636/1218212138871558204/1220294134145744896), but leaving open for discussion


**question in discord:**
>   Can you explain a little on the epoch shift?  if Im mathing correctly getEpochShift() is 89964 blocks i.e. ~5 days , so we shift the current blocks by 5 days?

**answer in discord:**
>    yes, by shifting blocks we make new epochs(week) start roughly on wednesday midnights. without shifting, new weeks start on a random time.
and by rounding timestamps we get the current epoch (week number)
we are aware this is not perfect and the start time may change as block generation can be inconsistent but we are ok with it.

As we can see this is not about startingPointWeek which is a value in magnitudes of weeks and deal with internal week accounting of the system. It is about timeshift which is a value in magnitudes of block, hence it is about the exact time where weeks are starting.
So the issue and discord comment provided are not related.


> An example provided by the researcher suggested a hypothetical scenario where the governance updates the starting point from 178 to 9. It's difficult to envision a scenario that justifies such an action. Following such a change, the internal week would adjust from 11 to 181 which is expected.

As provided by the test above, the problem lies in the fact that this function can only be callable by values less than internal week of system. So if the current week in the system is 11, it can only be executed with values less than 11. As I also mentioned before, this also overlaps with the creation of the contract in GovernanceFactory:
```solidity
      abi.encodeWithSelector(
        lockingImpl.__Locking_init.selector,
        address(mentoToken), /// @param _token The token to be locked in exchange for voting power in form of veTokens.
        startingPointWeek, ///   @param _startingPointWeek The locking epoch start in weeks. We start the locking contract from week 1 with min slope duration of 1
        0, ///                   @param _minCliffPeriod minimum cliff period in weeks.
        1 ///                    @param _minSlopePeriod minimum slope period in weeks.
      )
    );
    locking = Locking(address(lockingProxy));
```
As we can see from natspec comments, it is assumed that this variable is initiated with week 1 like min slope duration is initiated with 1 below.
Hence I don't think it is difficult to justify such action if we combine the fact that this function can only be executed with values less than current week, and also the code comments suggests same understanding.

> request poc
> 
> To faciliate discussion between watson and sponsor, but I am inclined to believe this issue is low severity based on sponsor comments, given admin ultimately decides which starting week is set.

The problem is like I explained, once the contract is created, setStartingPointWeek() function accept only the values that will completely broke the time system in the codebase. I would like to remind that the part where startingPointWeek is settled during contract deployment is done in "GovernanceFactory.sol" and it is in the scope, hence admin actually don't decide which starting week to set, it is already settled.

**nevillehuang**

@kosedogus Why would the protocol suddenly shift the starting point week from 178/180 --> 12/9? 

**kosedogus**

Sir that is the problem, it is not intended, during the creation of the contract this variable is assumed to start with 1, and also setter function approves that it can only be executed with value "0" after creation. But this variable is actually initiated with value "179", but setter still can only be executed with value "0". So the assumption is after 10 week passed, it should be callable with "9", and indeed it is only callable with values less than 10. But when it is called with values less than 10, because this variable is actually initiated with 179, internal week system is falling apart. So there is discrepancy between initialization and setter function. If the expected functionality is like in the setter function, then initialization should also be done accordingly and it should start with 1. If the expected functionality is like in the initialization, then setter function should be callable with values like 178 or 180.

**nevillehuang**

It seems there is indeed a inconsistency. I am inclined to think it would qualify as breaking core-contract functionality, given there is a non-zero chance of the scenario occurring, and the impact would break the internal time management which could potentially affect voting.

