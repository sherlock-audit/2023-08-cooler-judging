Shaggy Teal Blackbird

high

# `Clearinghouse` is still operationnal after `emergencyShutdown()`
## Summary
Users can still get loans and roll loans from the `Clearinghouse` even after it is deactivated by the `emergencyShutdown()` function

## Vulnerability Detail
The `Clearinghouse` is used by the protocol the fullfil loan requests of DAI against gOHM, funded by the treasury. That contract can be deactivated in case of emergency if an authorized address calls the `emergencyShutdown()` function.

```solidity
File Clearinghouse.sol

L360: function emergencyShutdown() external onlyRole("emergency_shutdown") {
        active = false;

        // If necessary, defund sDAI.
        uint256 sdaiBalance = sdai.balanceOf(address(this));
        if (sdaiBalance != 0) defund(sdai, sdaiBalance);

        // If necessary, defund DAI.
        uint256 daiBalance = dai.balanceOf(address(this));
        if (daiBalance != 0) defund(dai, daiBalance);

        emit Deactivated();
    }
```
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Clearinghouse.sol#L360C4-L360C5

This function sets the state variable `active` to `false` and withdraws the funds available in the contract.
However, the `active` variable is not checked when taking (`lendToCooler()`) or rolling (`rollLoan()`) a loan. This implies that if the `Clearinghouse` contract has been refilled by loan repaiment the `lendToCooler()` and `rollLoan()` are still active and people can still get loans out of the `Clearinghouse`.

## Impact
The `emergencyShutdown()` function does not deactivate the contract. User can still get loans when the contract is supposed to be deactivated.

### Proof of Concept
Here is a test that can be added to `Clearinghouse.t.sol` to showcase the issue:

```solidity
function test_lending_still_active_after_emergencyShutdown() public {
    // The rebalance() function is called, either because someone called lendToCooler()
    // or someone called rebalance() directly.
    clearinghouse.rebalance();

    // The clearinghouse is deactivated by calling emergencyShutdown()
    vm.prank(overseer);
    clearinghouse.emergencyShutdown();
    console.log("[+] clearinghouse has been shut down");
    
    // Borrowers are still able to repay their loans.
    // It might even happen more than usual because of people's panic

    // Simulate loan repayments, like in test "test_rebalance_deactivated_returnFunds"
    uint256 oneMillion = 1e24; // Random value, could be anything
    deal(address(sdai), address(clearinghouse), oneMillion);
    console.log("[+] clearinghouse received tokens from repayments");

    // Try to get a loan
    uint256 loanAmount_ = oneMillion; 
    (Cooler cooler, uint256 gohmNeeded, uint256 loanID) = _createLoanForUser(loanAmount_);
    console.log("[+] clearinghouse created a loan for user");

    // Loan has been taken succesfully
    assertEq(gohm.balanceOf(address(cooler)), gohmNeeded);
    assertEq(dai.balanceOf(address(user)), loanAmount_);
    assertEq(dai.balanceOf(address(cooler)), 0);

    // Move forward 1 day
    _skip(1 days);
    
    // Ensure user has enough collateral to roll the loan
    uint256 gohmExtra = cooler.newCollateralFor(loanID);
    _fundUser(gohmExtra);
    // Try to roll loan
    vm.prank(user);
    clearinghouse.rollLoan(cooler, loanID);
    console.log("[+] clearinghouse rolled a loan for user");

    assertEq(gohm.balanceOf(address(cooler)), gohmNeeded + gohmExtra);
}
```

## Code Snippet
https://github.com/ohmzeus/Cooler/blob/c6f2bbe1b51cdf3bb4d078875170177a1b8ba2a3/src/Clearinghouse.sol#L360-L372

## Tool used

Manual Review + Foundry

## Recommendation
Check the `active` status in `lendToCooler()` and `rollLoan()`.
```solidity
if (!active) revert NotActive();
```

