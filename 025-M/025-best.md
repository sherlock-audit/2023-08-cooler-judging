Old Chili Starling

medium

# Interest calculation in claimDefaulted in Clearinghouse is incorrect
## Summary
Interest calculation in ``claimDefaulted`` in ``ClearingHouse`` is incorrect resulting in ``outstandingDebt`` from ``TRSRY`` being incorrectly updated.

## Vulnerability Detail
In ``claimDefaulted``, keepers may batch ``claimDefaulted`` calls on Coolers. For each cooler/loan pair, defaulted debt and interest on the debt is accummulated to ``totalDebt`` and ``totalInterest`` respectively. The interest for a certain defaulted loan is calculated using [``interestFromDebt``](https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Clearinghouse.sol#L395-L399) where ``debt_`` is the defaulted debt amount which *includes interest*. This is flawed as interest on a loan (which is paid by the borrower, not lent using funds from ``TRSRY``) should be fixed and separate from the actual loaned amount (which uses ``TRSRY`` funds).

The ``totalInterest`` is the accumulation of these interest calculations and is used to update ``outstandingDebt`` which subtracts ``totalDebt - totalInterest``. The logic behind this is the *loaned* amount is subtracted from the treasury balance since the protocol no longer has hold over the lent DAI (the borrower has taken it in exchange for relinquishing collateral). As such the whole fixed interest calculated at origination should not be taken into consideration since it is not lent (and not TRSRY funds).

This issue is most apparent when a user has repaid the whole principal amount (but applies consistently for any repaid amount).

Consider the following example using the constant loan conditions in ``ClearingHouse`` (real numbers and no precision are used for simplicity):
``loan = 60000 DAI``
``interest = 5/1000 * 1/3  * 60000 = 100`` (taking duration as 1/3 of year for simplicity)

After borrower repays 600000 DAI, they default and ``claimDefaulted`` is called on their loan.
``debt = 600100 - 600000 = 100``
``interestFromDebt = 100 * 5/1000 * 1/3 = 1/6 > 0``
``totalDebt - totalInterest = 100 - 1/6 > 0``

``outstandingDebt`` is still reduced even though ``Clearinghouse`` has received the whole principal amount back, and so owns the whole original loaned amount.

If instead none of the debt is repaid,
``debt = 60100``
``interest = 5/1000 * 1/3 * 60100 = 100 + 1/6 > 100`` (this shows that interest calculated could actually be greater than the fixed interest calculated at origination leading to ``outstandingDebt`` being reduced by less than it should be).

In either case, the interest calculated is incorrect and will distort the update to ``outstandingDebt``.

## Impact
Debt accounting inaccuracies in ``claimDefaulted`` in ``ClearingHouse`` will accrue over time, causing undefined negative consequences in ``TRSRY`` accounting and the rest of the Olympus protocol (these areas are out of scope, but the main issue lies in ``ClearingHouse`` and I'm assuming accounting is significant, otherwise it wouldn't be updated).

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Clearinghouse.sol#L191-L245

## Tool used

Manual Review

## Recommendation
Consider introducing a separate field in ``Loan`` that caches the interest when clearing a request and use this for accounting for ``totalInterest``.