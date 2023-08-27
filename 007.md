Lone Chrome Armadillo

high

# Cooler.sol#collateralFor() - current configuration allows for collateral-less borrowing
## Summary
Both the Cooler contract as a single permissionless p2p and the ClearingHouse contract as a hardcoded DAI/gOHM Cooler, rely on the ``collateralFor()`` function to determine the borrow rates. Not checking the loan requested amount can lead to issues. 

## Vulnerability Detail
The function's formula is ``amount_ * (10 ** collateral().decimals())) / loanToCollateral_``.
In the case of the Olympus's intended usage, the decimals are 1e18, which cancel out with the 1e18 upscaling of the ``loanToCollateral_`` variable. Thus any amount that is lower than the provided ratio would round down to 0, ultimately leading to 0 collateral backing the loan, a.k.a theft of funds from the lender.
In the scenario that this occurs in an independent Cooler instance, the lender can just avoid clearing the request, but in the case of ClearingHouse using it, there is no check to make sure the user doesn't steal small DAI amounts from the treasury, through passing a small ``amount`` parameter to ClearingHouse's ``lendToCooler()``.

POC: Add this function to ClearingHouse.t.sol and run ``forge test -vvv``
```solidity
    function test_roundToZeroCollateral() public {
        (Cooler cooler, uint256 gohmNeeded, uint256 loanID) = _createLoanForUser(2999);
        assertEq(gohmNeeded, 0);
    }
```

## Impact
Direct theft of small amounts of DAI, which can accumulate to big drainage.

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L371-L373
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L129-L152

## Tool used

Manual Review

## Recommendation
Add a minimal amount of token check when creating a borrow request, or upscale the provided value by 1e18 and then round it back down when necessary.
