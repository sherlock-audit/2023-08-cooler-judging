High Champagne Blackbird

medium

# Unsafe call to `transfer/transferFrom` can result in stuck funds
## Summary
In the contracts `Clearinghouse`, the return values of ERC20 `transfer` and `transferFrom` are not checked to be true.

## Vulnerability Detail
The problem is that the transfer function from ERC20 returns a bool to indicate if the transfer was a success or not. As there are some tokens that do not revert on failure but instead return false and also `Clearinghouse` should work with all types of ERC20 tokens since it might be integrated with a protocol that does that, not checking the return value can result in tokens getting stuck. It is noted that `Cooler` contract uses `safeTransfer` and `safeTransferFrom` everywhere but `Clearinghouse` not practicing this.

## Impact
If a transfer call fails it will lead to stuck funds for a user. This only happens with a special class of ERC20 tokens though, so it is Medium severity.

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L140
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L178
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L241
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L319
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L356

## Tool used

Manual Review

## Recommendation
Use OpenZeppelin’s `SafeERC20` library and change transfer to `safeTransfer` and `transferFrom` to `safeTransferFrom`