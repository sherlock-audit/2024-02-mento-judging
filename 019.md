Proper Rose Duck

medium

# Discrepancy Between Initialization and Setter Function for "Internal Time" Resulting in Time Management Dysfunction

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