Brave Charcoal Ferret

high

# When debt and collateral have different decimals, the `collateralFor` calculation is incorrect, creating a malicious loan.
## Summary

## Vulnerability Detail
loanToCollateral is expressed in 10**debt().decimals(). (https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L95)

```solidity
function collateralFor(uint256 amount_, uint256 loanToCollateral_) public view returns (uint256) {
        return (amount_ * (10 ** collateral().decimals())) / loanToCollateral_;
    }
```
The logic of the function to find the collateral required to borrow an amount of debt tokens is as shown above.

Let's say the collateral is `USDC` (6 decimals) and the debt token is `WETH` (18 decimals).

If you lend `0.0005e18 WETH `per `1e6 USDC`, `loanToCollateral` is `500000000 * 10**18` (2000 USDC = 1 WETH).

So if we calculate the quantity of `USDC` to borrow `1 WETH` with the above value, we get 0.002 → `0` as `1e18 * (10 ** 6) / 500000000 * 10**18`.

We can exploit this to borrow 1 WETH without collateral.

## Impact
Can creating a malicious loan.
## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L371-L373
## Tool used

Manual Review

## Recommendation
Fix `collateralFor`.
```solidity
function collateralFor(uint256 amount_, uint256 loanToCollateral_) public view returns (uint256) {
        return (amount_ * (10 ** debt().decimals())) / loanToCollateral_;
    }
```