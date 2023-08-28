Jovial Oily Beetle

high

# Inaccurate Interest Calculation in `claimDefaulted` Function
## Summary
The `claimDefaulted` function in the contract uses the `interestFromDebt` function to calculate the interest that the protocol should have received upon repayment. The formula used in the `interestFromDebt` function does not accurately compute the target interest value, leading to discrepancies in the interest calculations.

## Vulnerability Detail
The current formula used in the 'interestFromDebt' function is:
$$\text{interest} = \text{debt} \times \text{interestPercent}$$ 
Where:
$$\text{interestPercent} = \left( \frac{\text{INTERESTRATE} \times \text{DURATION}}{365 \text{ days}} \right)$$

Using the provided example: 
- Amount: `1_000_000 * 1e18`
- INTEREST_RATE = `5e15 `(0.5%)
- Duration: `121 days`

Here the `loan.amount` will be equal to the requested amount + the interest
```javascript=254
# Cooler.sol
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
Therefore, the `loan.amount` will be equal to : `1,001,657,534,246,575,342,465,753`
```javascript=318
# Cooler.sol
    function claimDefaulted(uint256 loanID_) external returns (uint256, uint256, uint256) {
        Loan memory loan = loans[loanID_];
        delete loans[loanID_];

        if (block.timestamp <= loan.expiry) revert NoDefault();

        // Transfer defaulted collateral to the lender.
        collateral().safeTransfer(loan.lender, loan.collateral);

        // Log the event.
        factory().newEvent(loanID_, CoolerFactory.Events.DefaultLoan, 0);

        // If necessary, trigger lender callback.
        if (loan.callback) CoolerCallback(loan.lender).onDefault(loanID_, loan.amount, loan.collateral);
        return (loan.amount, loan.collateral, block.timestamp - loan.expiry);
    }
```

The calculated interest using the current formula is:
```javascript
# Clearinghouse.sol
            (uint256 debt, uint256 collateral, uint256 elapsed) = Cooler(coolers_[i]).claimDefaulted(loans_[i]);
            uint256 interest = interestFromDebt(debt);
```
```javascript
# Clearinghouse.sol
    function interestFromDebt(uint256 debt_) public pure returns (uint256) {
        uint256 interestPercent = (INTEREST_RATE * DURATION) / 365 days;
        return debt_ * interestPercent / 1e18;
    }
```
$$\text{interest} = 1,001,657,534,246,575,342,465,753 \times  \frac{5e15 }{1e18} \times \frac{121}{365}$$
This results in: 
$$\text{interest} = 1,660,281,666,353,912,553,950$$ 
However, this does not match the expected interest of `1,657,534,246,575,342,465,753` as there is an difference of `2,747,419,778,570,088,197` between the two values.




This error is due to the fact that the used formula is designed to calculate the interest that will be accumulated on a specific debt, not to calculate how much interest was collected from a total of debt+interest.

The correct formula that should be used to calculate the accumulated interest is the following :
$$\text{interest} = \text{debt} \times \left( 1 - \frac{1}{1 + \frac{\text{interestPercent}}{1e18}} \right)$$

Using the correct formula:
$$\text{interest} = 1,001,657,534,246,575,342,465,753 \times \left( 1 - \frac{1}{1 + \frac{5e15 }{1e18}  \times \frac{121}{365}} \right)$$
This results in: $$\text{interest} = 1,657,534,246,575,342,465,753$$

This showcases the discrepancy between the current and the correct interest.


## Impact
The inaccurate calculation can lead to financial discrepancies in the protocol, due to relying on an incorrect collected interest value.

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L254-L265
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L318-L333
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L204-L205
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L395-L398

## Tool used
Manual Review

## Recommendation
Modify the `interestFromDebt` function to use the correct formula:
$$\text{interest} = \text{debt} \times \left( 1 - \frac{1}{1 + \frac{\text{interestPercent}}{1e18}} \right)$$