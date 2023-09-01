Mythical Honey Gecko

high

# Emergency shutdown is ineffective within 1 week if someone has previously called the rebalance function
## Summary

The current implementation of the `Clearinghouse.sol` contains a logical issue that could block the `emergencyShutdown()` function from successfully defunding the contract when it needs to be deactivated in an emergency.
## Vulnerability Detail

The `rebalance()` function has a condition that checks whether `fundTime > block.timestamp` before proceeding. If the condition is met, the function returns `false`, effectively blocking the rebalancing (and defunding in case the contract is inactive).

```solidity
function rebalance() public returns (bool) {
        // If the contract is deactivated, defund.  
        uint256 maxFundAmount = active ? FUND_AMOUNT : 0;        
        // Update funding schedule if necessary.
        if (fundTime > block.timestamp) return false;  
        fundTime += FUND_CADENCE;
```

The problem is when `emergencyShutdown()` is used to deactivate the contract 
```solidity
/// @notice Deactivate the contract and return funds to treasury.
    function emergencyShutdown() external onlyRole("emergency_shutdown") {
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
This function  deactivate the contract and return the necessary funds to the treasury. 

We go back to the rebalance function. Already at the beginning of the rebalance function we have a comment: 
```solidity
// If the contract is deactivated, defund.
```
But the defund logic is implemented at the end of the contract.
If an emergency arises just after a rebalance call, and before the next scheduled rebalance time(7 days), the system will be prevented from returning funds to the treasury due to the mentioned time check, potentially leaving the funds exposed to risks for an extended period.

```solidity
if (fundTime > block.timestamp) return false;
```
## Impact

The delay in defunding due to the timing constraints might leave the assets in the contract exposed to whatever risks prompted the emergency shutdown. This will lead to potential financial losses.

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L276-L281
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L359-L361

## Tool used

Manual Review

## Recommendation

Defund logic must shift before `fundTime` check.