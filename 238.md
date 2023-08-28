Generous Juniper Mink

high

# `rollLoan` requests collateral even for the interest of the loan
## Summary
`rollLoan` requests collateral even for the interest of the loan 

## Vulnerability Detail
Upon accepting a request, the collateral for it is calculated solely on the borrow amount.  
```solidity
        uint256 collat = collateralFor(req.amount, req.loanToCollateral);
```
However, when storing the `loan.amount` it saves the value of the borrow amount + the interest.
```solidity
loans.push(
            Loan({
                request: req,
                amount: req.amount + interest,
                unclaimed: 0,
                collateral: collat,
                expiry: expiration,
                lender: msg.sender,
                repayDirect: repayDirect_,
                callback: isCallback_
            })
        );
```
This means that when calling `rollLoan` to calculate the new collateral value needed, it uses the `loan.amount` value, which also consists of interest, meaning the interest of the loan will also have to be backed up by collateral.
```solidity
        uint256 newCollateral = newCollateralFor(loanID_);
```
```solidity 
    function newCollateralFor(uint256 loanID_) public view returns (uint256) {
        Loan memory loan = loans[loanID_];
        // Accounts for all outstanding debt (borrowed amount + interest).
        uint256 neededCollateral = collateralFor(
            loan.amount,
            loan.request.loanToCollateral
        );

        return
            neededCollateral > loan.collateral ?
            neededCollateral - loan.collateral :
            0;
    }
```

## Impact
User will be forced into overcollateralizing their borrow

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L377C1-L389C6
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L199

## Tool used

Manual Review

## Recommendation
Do not request for the interest to be backed up by collateral.