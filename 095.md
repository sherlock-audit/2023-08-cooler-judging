Cold Rusty Chameleon

high

# Insufficient checks on transferOwnership can make important operations revert.
## Summary
A Lender transfers loan ownership to an address, which is not checked if it implements CollerCallback, making ```repayLaon()```, ```rollLoan()``` and ```claimDefaulted()``` revert causing Borrower loan to default and inconvenience to both Lender and Borrower. 

## Vulnerability Detail
To transfer the ownership of a loan, following steps are required.
1 - A **Lender** calls ```approveTransfer()``` to allow an address to take ownership of a particular loan.
2 - Approved address calls ```transferOwnership()``` to take ownership of the loan.

However, it is assumed that the approved address always implements **CoolerCallback** in case of ```loan.callback == true```. But in reality, this is not the case, as the approved address can be an EOA.

If the ownership of such a loan where ```loan.callback==true``` is claimed by an EOA, ```repayLaon()```, ```rollLoan()``` and ```claimDefaulted()``` will always revert as they expect the Lender to implement CoolerCallack.

References:
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L351
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L185
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L216
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L331

## Impact
1 - Failure of ```repayLoan()``` can cause the loan to expire, making the loan defaulted. Although, the lender can again transfer ownership to a working Lender implementation on borrower's request, but there can be 2 issues.
a)-  In a decentralized world, how can the borrower contact the **Lender** timely to ask to transfer the ownership to a working implementation to avoid default.
b) - **Lender** sees that loan default is more profitable for him so he does not transfer the ownership to a working implementation, time passes, loan is expired, lender transfers ownership to a working implementation, calls ```claimDefaulted()```,makes profit, borrower loses collateral tokens.
2 - Other important operations like ```rollLoan()``` and ```claimDefaulted()``` will not work causing issues to both Borrower and Lender. 

## Code Snippet
```solidity
    function transferOwnership(uint256 loanID_) external {
        if (msg.sender != approvals[loanID_]) revert OnlyApproved();

        // Update the load lender.
        loans[loanID_].lender = msg.sender;
        // Clear transfer approvals.
        approvals[loanID_] = address(0);
    }
```

## Tool used

Manual Review

## Recommendation
In transferOwnership(), if the loan.callback == true, check that msg.sender implements CoolerCallback as happening in [clearRequest()](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L241) to make sure loan's new owner implements CoolerCallback.
