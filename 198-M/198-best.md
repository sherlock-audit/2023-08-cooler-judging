Virtual Punch Raven

false

# An attacker can manipulate the interest_, loanToCollateral_, duration_ parameters, bypassing Clearinghouse.rollLoan()
## Summary

The provideNewTermsForRoll() function (https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Cooler.sol#L282-L300) is called in Clearinghouse.rollLoan() (https://github.com/ohmzeus/Cooler /blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Clearinghouse.sol#L165). However, an attacker can introduce their interest_, loanToCollateral_, duration_ parameters, bypassing the Clearinghouse.rollLoan() call. An attacker can directly call Cooler.provideNewTermsForRoll() directly with their input parameters.

function provideNewTermsForRoll(
     uint256 loanID_,
     uint256 interest_,
     uint256 loanToCollateral_,
     uint256 duration_
) external {
     Loan storage loan = loans[loanID_];

     if (msg.sender != loan.lender) revert OnlyApproved();

     loan request =
         Request(
             loan.amount,
             interest_,
             loanToCollateral_,
             duration_,
             true
         );
}

provideNewTermsForRoll() is of type external and there is no check of who is calling the function other than what the owner of the given loanID_ is calling.

This error will result in invalid interest_, loanToCollateral_, duration_ data.

## Tool used

Manual Review

## Recommendation

Check in Cooler.provideNewTermsForRoll() msg.sender == Clearinghouse.sol