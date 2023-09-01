Generous Juniper Mink

high

# `rollLoan` creates compound interest
## Summary
After accepting new terms via `rollLoan`, the interest from the original borrow accrues interest too (compound interest)

## Vulnerability Detail
After a loan is active, the lender can propose new terms, which can then be accepted via `rollLoan`. The problem is that when calculating the needed interest, it is based off of `loan.amount` which consists of the original borrow + interest, meaning that the interest will also accrue interest and result into compound interest. 
```solidity
uint256 newDebt = interestFor(loan.amount, loan.request.interest, loan.request.duration); // @audit loan.amount contains the original interest 

        // Update memory accordingly.
        loan.amount += newDebt;
```

## Impact
User will overpay interest 

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L200C9-L203C32


## Tool used

Manual Review

## Recommendation
When creating a loan, account the interest and the borrow separately, so upon accepting new terms ,there isn't unneeded compound interest calculation 
