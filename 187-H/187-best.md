Rough Garnet Ape

high

# isCoolerCallback can be bypassed
## Summary

The lender can bypass `CoolerCallback.isCoolerCallback()` validation without implements the `CoolerCallback` abstract.

In the provided example, this may force the loan to default.

## Vulnerability Detail

The `CoolerCallback.isCoolerCallback()` is intended to ensure that the lender implements the `CoolerCallback` abstract at line 241 when the parameter `isCallback_` is `true`.

```solidity=!
function clearRequest(
    uint256 reqID_,
    bool repayDirect_,
    bool isCallback_
) external returns (uint256 loanID) {
    Request memory req = requests[reqID_];

    // If necessary, ensure lender implements the CoolerCallback abstract.
    if (isCallback_ && !CoolerCallback(msg.sender).isCoolerCallback()) revert NotCoolerCallback();

    // Ensure loan request is active. 
    if (!req.active) revert Deactivated();

    // Clear the loan request in memory.
    req.active = false;

    // Calculate and store loan terms.
    uint256 interest = interestFor(req.amount, req.interest, req.duration);
    uint256 collat = collateralFor(req.amount, req.loanToCollateral);
    uint256 expiration = block.timestamp + req.duration;
    loanID = loans.length;
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

    // Clear the loan request storage.
    requests[reqID_].active = false;

    // Transfer debt tokens to the owner of the request.
    debt().safeTransferFrom(msg.sender, owner(), req.amount);

    // Log the event.
    factory().newEvent(reqID_, CoolerFactory.Events.ClearRequest, 0);
}
```
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L233-L275

However, this function doesn't provide any protection. The lender can bypass this check without implementing the `CoolerCallback` abstract by calling the `Cooler.clearRequest()` function using a contract that implements the `isCoolerCallback()` function and returns a `true` value.

For example:

```solidity=!
contract maliciousLender {
    function isCoolerCallback() pure returns(bool) {
        return true;
    }
    
    function operation(
        address _to,
        uint256 reqID_
    ) public {
        Cooler(_to).clearRequest(reqID_, true, true);
    }
    
    function onDefault(uint256 loanID_, uint256 debt, uint256 collateral) public {}
}
```

By being the `loan.lender` with implement only `onDefault()` function, this will cause the `repayLoan()` and `rollLoan()` methods to fail due to revert at `onRepay()` and `onRoll()` function. The borrower cannot repay and the loan will be defaulted.

After the loan default, the attacker can execute `claimDefault()` to claim the collateral.

Furthermore, there is another method that allows lenders to bypass the `CoolerCallback.isCoolerCallback()` function which is loan ownership transfer.

```solidity=!
/// @notice Approve transfer of loan ownership rights to a new address.
/// @param  to_ address to be approved.
/// @param  loanID_ index of loan in loans[].
function approveTransfer(address to_, uint256 loanID_) external {
    if (msg.sender != loans[loanID_].lender) revert OnlyApproved();

    // Update transfer approvals.
    approvals[loanID_] = to_;
}

/// @notice Execute loan ownership transfer. Must be previously approved by the lender.
/// @param  loanID_ index of loan in loans[].
function transferOwnership(uint256 loanID_) external {
    if (msg.sender != approvals[loanID_]) revert OnlyApproved();

    // Update the load lender.
    loans[loanID_].lender = msg.sender;
    // Clear transfer approvals.
    approvals[loanID_] = address(0);
}
```

Normally, the lender who implements the `CoolerCallback` abstract may call the `Cooler.clearRequest()` with the `_isCoolerCallback` parameter set to `true` to execute logic when a loan is repaid, rolled, or defaulted.

But the lender needs to change the owner of the loan, so they call the `approveTransfer()` and `transferOwnership()` functions to the contract that doesn't implement the `CoolerCallback` abstract (or implement only `onDefault()` function to force the loan default), but the `loan.callback` flag is still set to `true`.

Thus, this breaks the business logic since the three callback functions don't need to be implemented when the `isCoolerCallback()` is set to `true` according to the dev note in the `CoolerCallback` abstract below:

> /// @notice Allows for debt issuers to execute logic when a loan is repaid, rolled, or defaulted.
/// @dev    The three callback functions must be implemented if `isCoolerCallback()` is set to true.

## Impact

1. The lender forced the Loan become default to get the collateral token, owner lost the collateral token.

2. Bypass the `isCoolerCallback` validation.
## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L241

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L338-L343

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Cooler.sol#L347-L354

## Tool used

Manual Review

## Recommendation

Only allowing callbacks from the protocol-trusted address (eg., `Clearinghouse` contract).

Disable the transfer owner of the loan when the `loan.callback` is set to `true`.