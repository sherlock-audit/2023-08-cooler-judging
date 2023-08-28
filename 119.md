Wobbly Felt Rook

medium

# At claimDefaulted, the lender may not receive the token because the Unclaimed token is not processed
## Summary

`claimDefaulted` does not handle `loan.unclaimed`  . This preventing the lender from receiving the debt repayment.

## Vulnerability Detail

```solidity
function claimDefaulted(uint256 loanID_) external returns (uint256, uint256, uint256) {
  Loan memory loan = loans[loanID_];
  delete loans[loanID_];
```

 Loan data is deletead in `claimDefaulted` function. `loan.unclaimed` is not checked before data deletead. So, if `claimDefaulted` is called while there are unclaimed tokens, the lender will not be able to get the unclaimed tokens.

## Impact

Lender cannot get unclaimed token.

## Code Snippet

[https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Cooler.sol#L318-L320](https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Cooler.sol#L318-L320)

## Tool used

Manual Review

## Recommendation

Process unclaimed tokens before deleting loan data.

```diff
function claimDefaulted(uint256 loanID_) external returns (uint256, uint256, uint256) {
+   claimRepaid(loanID_)
    Loan memory loan = loans[loanID_];
    delete loans[loanID_];
```