Rich Denim Barracuda

high

# calculateEmission emit less token due to precision loss

## Summary
calculateEmission emit less token due to presion loss.

## Vulnerability Detail
In [Emission.sol#L103-L125](https://github.com/sherlock-audit/2024-02-mento/blob/main/mento-core/contracts/governance/Emission.sol#L103-L125), it uses an approximation formula to calculate releasable token amount.

```solidity
E(t) = supply * (1 - (t / A) + (t^2 / 2A^2) - (t^3 / 6A^3) + (t^4 / 24A^4))
```

However, it use a gas-save way to implement this calculation in solidity code.
```solidity
    uint256 term1 = (SCALER * t) / A;
    uint256 term2 = (term1 * t) / (2 * A);
    uint256 term3 = (term2 * t) / (3 * A);
    uint256 term4 = (term3 * t) / (4 * A);
    uint256 term5 = (term4 * t) / (5 * A);
```

It do division before multiplication when calculating `(t^2 / 2A^2)`, `(t^3 / 6A^3)` `(t^4 / 24A^4)`, (t^5 / 120A^5), which has precision loss.

The larger the `t` is, the delta caused by precision loss also getting larger. Here is a simulation script in python.

```python
A = 454968308
SCALER = int(1e18)

def buggy(t):
    term1 = (SCALER * t) // A
    term2 = (term1 * t) // (2 * A)
    term3 = (term2 * t) // (3 * A)
    term4 = (term3 * t) // (4 * A)
    term5 = (term4 * t) // (5 * A)
    return (SCALER + term2 + term4, term1 + term3 + term5)

def mul_before_div(t):
    term1 = (SCALER * t) // A
    term2 = (t * t) // (2 * A * A)
    term3 = (t * t * t) // (6 * A * A * A)
    term4 = (t * t * t * t) // (24 * A * A * A * A)
    term5 = (t * t * t * t * t) // (120 * A * A * A * A * A)
    return (SCALER + term2 + term4, term1 + term3 + term5)

for i in range(100):
    left = buggy(i)
    right = mul_before_div(i)
    l_diff = left[0] - left[1]
    r_diff = right[0] - right[1]
    print(f"t: {i} delta: {l_diff - r_diff}")
```

```solidity
t: 0 delta: 0
t: 1 delta: 2
t: 2 delta: 9
t: 3 delta: 21
t: 4 delta: 38
t: 5 delta: 60
t: 6 delta: 86
t: 7 delta: 118
t: 8 delta: 154
t: 9 delta: 195
t: 10 delta: 241
t: 11 delta: 292
t: 12 delta: 347
t: 13 delta: 408
t: 14 delta: 473
t: 15 delta: 543
t: 16 delta: 618
t: 17 delta: 698
t: 18 delta: 782
t: 19 delta: 871
t: 20 delta: 966
t: 21 delta: 1065
t: 22 delta: 1169
t: 23 delta: 1277
t: 24 delta: 1391
t: 25 delta: 1509
t: 26 delta: 1632
t: 27 delta: 1760
t: 28 delta: 1893
t: 29 delta: 2031
t: 30 delta: 2173
```

```solidity
delta = buggy_implementation - mul_before_div_implementation
```

## Impact
This function returns emit token amount. 
```solidity
    uint256 scheduledRemainingSupply = (emissionSupply * (positiveAggregate - negativeAggregate)) / SCALER;
    amount = emissionSupply - scheduledRemainingSupply - totalEmittedAmount;
```

In current buggy version:  
`t` increase 
=> `(positiveAggregate - negativeAggregate)` increase 
=> `scheduledRemainingSupply` increase 
=> `amount` decrease

emitTokens will emit less token then expected. And it will spend less time to getting decay.

## Code Snippet
```solidity
  function calculateEmission() public view returns (uint256 amount) {
    uint256 t = (block.timestamp - emissionStartTime);

    // slither-disable-start divide-before-multiply
    uint256 term1 = (SCALER * t) / A;
    uint256 term2 = (term1 * t) / (2 * A);
    uint256 term3 = (term2 * t) / (3 * A);
    uint256 term4 = (term3 * t) / (4 * A);
    uint256 term5 = (term4 * t) / (5 * A);
    // slither-disable-end divide-before-multiply

    uint256 positiveAggregate = SCALER + term2 + term4;
    uint256 negativeAggregate = term1 + term3 + term5;

    // Avoiding underflow in case the scheduled amount is bigger than the total supply
    if (positiveAggregate < negativeAggregate) {
      return emissionSupply - totalEmittedAmount;
    }

    uint256 scheduledRemainingSupply = (emissionSupply * (positiveAggregate - negativeAggregate)) / SCALER;

    amount = emissionSupply - scheduledRemainingSupply - totalEmittedAmount;
  }
```

## Tool used

Manual Review

## Recommendation
Do calculation in a multiplication before div way, like this python: 
```python
def mul_before_div(t):
    term1 = (SCALER * t) // A
    term2 = (t * t) // (2 * A * A)
    term3 = (t * t * t) // (6 * A * A * A)
    term4 = (t * t * t * t) // (24 * A * A * A * A)
    term5 = (t * t * t * t * t) // (120 * A * A * A * A * A)
    return (SCALER + term2 + term4, term1 + term3 + term5)
```