Rhythmic Gingerbread Walrus

medium

# Direct Call To The rollLoan Function Might Make The Loan Impossible To Be Fulfilled
## Summary

A loan can be "rolled" to new terms inside the contract Cooler.sol in the function `rollLoan` . It updates the memory accordingly 
and rolls the loan to new terms provided the it is active.


## Vulnerability Detail

1.)  Confirming from the sponsor , the author made an assumption that prior to calling the `rollLoan` function (inside Cooler.sol) a call to `Clearinghouse.rollLoan` has already occurred , which would clear the loan request.

2.) But if a direct call happens to the `rollLoan`  (accidentally or attacker does it with the right incentives), then the loan gets rolled plus at L206 we do `loan.request.active = false;` which
sets the loan to false. Due to this the loan can never be fulfilled due to the check at L244 i.e.

`if (!req.active) revert Deactivated();`

There is also one more "assumption" made , i.e. `provideNewTermsForRoll` has been called prior , similar fix to the above will suffice for this.


## Impact

The loan can never be fulfilled now as `rollLoan` was called directly. An attacker can call rollLoan at any active loan,  provided the new collateral paid by the attacker makes sense , so that the attacker can make the loan useless(can't be fulfilled by a lender)

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L192

## Tool used

Manual Review

## Recommendation

Make sure to call `Clearinghouse.rollLoan` prior by hardcoding the call inside the `rollLoan` function.