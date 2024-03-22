Dapper Honey Raccoon

medium

# LockingBase storage layout is not aligned on a 50-basis

## Summary

`LockingBase`'s storage is not 50-slots aligned.

This bad practice may lead to issues during future upgrades.

## Vulnerability Detail

Storage slots for upgradeable contracts should be aligned on 50-slots.
If a contract needs 60 slots for example, it should reserve 100 slots.

During future upgrades, this 50-slot alignment allows to reduce the risk of data overwriting due to a mistake.

## Impact

Future upgrades may write data at the wrong location due to inconsistency in storage layout.

**Low** impact

## Code Snippet

```solidity
abstract contract LockingBase is OwnableUpgradeable, IVotesUpgradeable {
  uint32 public constant WEEK = 120_960;
  
  uint32 constant MAX_CLIFF_PERIOD = 103;
  
  uint32 constant MAX_SLOPE_PERIOD = 104;
  
  uint32 constant ST_FORMULA_BASIS = 1 * (10**8);
  
  IERC20Upgradeable public token; // @POC: 1
  
  uint256 public counter; // @POC: 2
  
  bool public stopped; // @POC: 3
  
  uint256 public minCliffPeriod; // @POC: 4
  
  uint256 public minSlopePeriod; // @POC: 5
  
  uint256 public startingPointWeek; // @POC: 6
  
  struct Lock {
    address account;
    address delegate;
  }
  
  mapping(uint256 => Lock) locks; // @POC: 7

  struct Account {
    LibBrokenLine.BrokenLine balance;
    LibBrokenLine.BrokenLine locked;
    uint96 amount;
  }

  mapping(address => Account) accounts; // @POC: 8
  
  LibBrokenLine.BrokenLine public totalSupplyLine; // @POC: 9, 10, 11, 12

  // @POC: REDACTED

  uint256[50] private __gap; // @POC: __gap should not be 50
}
```

## Tool used

Manual Review

## Recommendation

Consider modifying the `__gap` size.

After several tests, the right value is `38`.

```solidity
  uint256[38] private __gap; 
```

