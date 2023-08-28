Gorgeous Hazel Mantaray

medium

# DoS attack possible in request loan token.
## Summary
DoS attack possible in request loan token.

## Vulnerability Detail
requestLoan() function doesn't check input parameters so if someone wants to attack the pool and stop requests, can spam this function with parameters that return zero in collateralFor() function. we know many erc20 accept zero transfer and safeTransfer doesn't have a check for zero amount, so this is possible to request so many loans and spend so many gas fees to denial of service the requestLoan() function. Because this function needs to push so many requests in the Request[ ] array, nobody is able to request a loan until the attacker spammed requests get finished.

## Impact
DoS'ing the requestLoan() function.

## Code Snippet
```solidity 

function requestLoan(
        uint256 amount_,
        uint256 interest_,
        uint256 loanToCollateral_,
        uint256 duration_
    ) external returns (uint256 reqID) {
        reqID = requests.length;
        requests.push(
            Request({
                amount: amount_,
                interest: interest_,
                loanToCollateral: loanToCollateral_,
                duration: duration_,
                active: true
            })
        );

        // The collateral is taken upfront. Will be escrowed
        // until the loan is repaid or defaulted.
        collateral().safeTransferFrom(
            msg.sender,
            address(this),
            collateralFor(amount_, loanToCollateral_)
        );

        // Log the event.
        factory().newEvent(reqID, CoolerFactory.Events.RequestLoan, 0);
    }
```
https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Cooler.sol#L98-L125
## Tool used

Manual Review

## Recommendation
Consider checking the return value of collateralFor() and ensure that is not a zero amount.
