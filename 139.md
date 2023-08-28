Old Chili Starling

high

# Anyone can increase a borrower's owed amount for loans made using ``Clearinghouse``
## Summary
Anyone can call ``rollLoan`` on a borrower's ``Clearinghouse`` loan to increase their debt without providing extra collateral. (the core issue for this lies in the ``Clearinghouse``, not in ``Cooler`` since only the ``Clearinghouse`` can update loans provided by itself)

## Vulnerability Detail
``Clearinghouse`` simplifies the process of rolling loans through the function [``rollLoan``](https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Clearinghouse.sol#L160-L184) which can be called by anyone on any loan provided by ``Clearinghouse``. This allows anyone to increased the debt of ``Clearinghouse`` borrowers without providing extra collateral, granted the remaining debt is lower than the initial principal.

Consider the following example (using real numbers for simplicity):
- Alice takes a loan of 3000 DAI from ``Clearinghouse``, depositing 1 gOHM of collateral. Her current loan amounts:
``interest = 5/1000 * 1/3 * 3000 = 5`` (1/3 approximates 121/365 for simplicity)
``debt = 3000 + 5 = 3005``
- A few days later, Alice repays 605 DAI, so ``debt = 3005 - 605 = 2400``
- Bob calls ``rollLoan`` on Alice's loan
New loan amounts:
``newCollateral = 2400 / 3000 = 0.8 <= 1`` so ``newCollateral = 0`` and Bob does not pay extra collateral
``newInterest = 5/1000 * 1/3 * 2400 = 4``
``loan.amount = 2400 + 4 = 2404``
``loan.collateral = 1 + 0 = 1``

Bob can call this without extra collateral as long as ``loan.amount < 3000`` (though if this was to grief the attacker probably wouldn't call this a large number of times due to increasing the loan duration by a 121 days each time). Alice is now forced to pay an extra 4 DAI to regain her collateral.

The loss of funds depends on the borrowed amount and remaining amount when the attacker calls ``rollLoan``.

## Impact
Loss of funds for borrower due to increased debt amount to be paid to retrieve collateral.

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Clearinghouse.sol#L160-L184

## Tool used

Manual Review

## Recommendation
Consider validating that ``msg.sender`` is the input Cooler's ``owner()``.