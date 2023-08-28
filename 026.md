Spicy Brown Chipmunk

high

# Lender is able to steal borrowers collateral by calling rollLoan  with unfavourable terms on behalf of the borrower.
## Summary
A Lender is able to call provideNewTermsForRoll with whatever terms they want and then can call rollLoan on behalf of the borrower forcing them to roll the loan with the terms they provided. They can abuse this to make the loan so unfavourable for the borrower to repay that they must forfeit their collateral to the lender.

## Vulnerability Detail
Say a user has 100 collateral tokens valued at $1,500 and they wish to borrow 1,000 debt tokens valued at $1,000 they would would call: (values have simplified for ease of math)
```Solidity
requestLoan("1,000 debt tokens", "5% interest", "10 loan tokens for each collateral", "1 year")
```
If a lender then clears the request the borrower would expect to have 1 year to payback 1,050 debt tokens to be able to receive their collateral back.

However a lender is able to call provideNewTermsForRoll with whatever terms they wish: i.e. 
```Solidity
provideNewTermsForRoll("loanID", "10000000% interest", "1000 loan tokens for each collateral" , "1 year")
```
They can then follow this up with a call to rollLoan(loanID):
During the rollLoan function  the interest is recalculated using:
```Solidity
    function interestFor(uint256 amount_, uint256 rate_, uint256 duration_) public pure returns (uint256) {
        uint256 interest = (rate_ * duration_) / 365 days;
        return (amount_ * interest) / DECIMALS_INTEREST;
    }
```
As rate_ & duration_ are controllable by the borrower when they call provideNewTermsForRoll they can input a large number that the amount returned is much larger then the value of the collateral.  i.e. input a rate_ of amount * 3 and duration of 365 days so that the interestFor returns 3,000.

This amount gets added to the existing [loan.amount](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L203) and would make it too costly to ever repay as the borrower would have to spend more then the collateral is worth to get it back. i.e. borrower now would now need to send 4,050 debt tokens to receive their $1,500 worth of collateral back instead of the expected 1050.

The extra amount should result in more collateral needing to be sent however it is calculated using loan.request.loanToCollateral which is also controlled by the lender when they call provideNewTermsForRoll,  allowing them to input a value that will result in newCollateralFor returning 0 and no new collateral needing to be sent.

```Solidity
    function newCollateralFor(uint256 loanID_) public view returns (uint256) {
        Loan memory loan = loans[loanID_];
        // Accounts for all outstanding debt (borrowed amount + interest).
        uint256 neededCollateral = collateralFor(loan.amount, loan.request.loanToCollateral);  
        // Lender can force neededCollateral to always be less than loan.collateral

        return neededCollateral > loan.collateral ? neededCollateral - loan.collateral : 0;
    }
```
As a result a borrower who was expecting to have repay 1050 tokens to get back their collateral may now need to spend many multiples more of that and will just be forced to just forfeit their collateral to the lender. 


## Impact
Borrower will be forced to payback the loan at unfavourable terms or forfeit their collateral.

## Code Snippet
[Cooler.sol#L192-L217](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L192-L217)
[Cooler.sol#L282-L300](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L282-L300)

## Tool used
Manual Review

## Recommendation
Add a check restricting rollLoan to only be callable by the owner. i.e.:
```Solidity
function rollLoan(uint256 loanID_) external {
        Loan memory loan = loans[loanID_];
        
        if (msg.sender != owner()) revert OnlyApproved();
```
Note: unrelated but rollLoan is also missing its event should add:
```Solidity
factory().newEvent(reqID_, CoolerFactory.Events.RollLoan, 0);
```