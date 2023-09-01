Steep Bamboo Elk

high

# Malicious lender can use Callbacks to create Loan that cannot be repaid
## Summary
## Vulnerability Detail

The callback function, described by code comments:

```solidity
// Allows for debt issuers to execute logic when a loan is repaid, rolled, or defaulted.
```

A Lender can add a Callback to Loan which always reverts for `onRepay()`. 

```solidity
    function onRepay(uint256 loanID_, uint256 amount_) external { 
        if(!factory.created(msg.sender)) revert OnlyFromFactory();
        _onRepay(loanID_, amount_); // **<------ ADD REVERTING HOOK**
    }
```

When a borrower calls `repayLoan`, the callback  will called

```solidity
if (loan.callback) CoolerCallback(loan.lender).onRepay(loanID_, repaid_);
```

and since the lender set it to always revert, the repay will revert, ultimately leading to them never being able to repay their loan and losing funds.

## Impact
Malicious lender can give loans which are impossible to repay, and thus force them to default and seize their collateral

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/CoolerCallback.sol#L30-L33

## Tool used

Manual Review

## Recommendation

Do not allow hooks for onRepay(). This limits the functionality but is one way to stop this attack.
