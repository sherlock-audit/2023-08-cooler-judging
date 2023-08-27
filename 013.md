Rough Arctic Giraffe

medium

# Inadequate Debt Adjustment in rebalance Function Undermines Clearinghouse Financial Integrity
## Summary
when the amount of DAI held in the contract exceeds the maximum fund amount `maxFundAmount.` In this case, the contract attempts to defund the excess DAI by adjusting the debt owed to the `TRSRY,` the calculation to adjust the debt doesn't consider whether the current debt is sufficient to cover the adjustment.
## Vulnerability Detail
here is the vulnerable part in code :
```solidity
// @return False if too early to rebalance. Otherwise, true.
    function rebalance() public returns (bool) {
        // If the contract is deactivated, defund.
        uint256 maxFundAmount = active ? FUND_AMOUNT : 0;        
        // Update funding schedule if necessary.
        if (fundTime > block.timestamp) return false;
        fundTime += FUND_CADENCE;

        uint256 daiBalance = sdai.maxWithdraw(address(this));
        uint256 outstandingDebt = TRSRY.reserveDebt(dai, address(this));
        // Rebalance funds on hand with treasury's reserves.
        if (daiBalance < maxFundAmount) {
            // Since users loans are denominated in DAI, the clearinghouse
            // debt is set in DAI terms. It must be adjusted when funding.
            uint256 fundAmount = maxFundAmount - daiBalance;
            TRSRY.setDebt({
                debtor_: address(this),
                token_: dai,
                amount_: outstandingDebt + fundAmount
            });

            // Since TRSRY holds sDAI, a conversion must be done before
            // funding the clearinghouse.
            uint256 sdaiAmount = sdai.previewWithdraw(fundAmount);
            TRSRY.increaseWithdrawApproval(address(this), sdai, sdaiAmount);
            TRSRY.withdrawReserves(address(this), sdai, sdaiAmount);

            // Sweep DAI into DSR if necessary.
            uint256 idle = dai.balanceOf(address(this));
            if (idle != 0) _sweepIntoDSR(idle);
        } else if (daiBalance > maxFundAmount) {
            // Since users loans are denominated in DAI, the clearinghouse
            // debt is set in DAI terms. It must be adjusted when defunding.
            uint256 defundAmount = daiBalance - maxFundAmount;
            TRSRY.setDebt({
                debtor_: address(this),
                token_: dai,
                amount_: (outstandingDebt > defundAmount) ? outstandingDebt - defundAmount : 0
            });

            // Since TRSRY holds sDAI, a conversion must be done before
            // sending sDAI back.
            uint256 sdaiAmount = sdai.previewWithdraw(defundAmount);
            sdai.approve(address(TRSRY), sdaiAmount);
            sdai.transfer(address(TRSRY), sdaiAmount);
        }
        return true;
    }

    /// @notice Sweep excess DAI into vault.
```
- so If the contract holds more DAI than the maximum fund amount, it will attempt to adjust the debt owed to the Treasury, if the outstanding debt is less than the calculated `defundAmount,` the debt adjustment calculation won't reduce the debt properly. This will result in an incorrect debt value stored in the contract.
and The Clearinghouse relies on accurate debt calculations to ensure that the total debt obligations are properly maintained. Incorrect debt calculations can lead to inaccurate financial records, potentially resulting in mismatched balances and disrupted  accounting.

- Here a Real Scenario:
- The `Clearinghouse` holds 20,000 DAI in its balance.
- The `maxFundAmount` is set to 10,000 DAI.
- The current outstanding debt owed to the Treasury is 15,000 DAI.
the scenario containe:
- The contract holds an excess of `20,000 DAI - 10,000 DAI = 10,000 DAI`.
- The contract will attempt to adjust the debt by calculating `defundAmount` as `10,000` DAI, the outstanding debt is only 15,000DAI.

- so the issue manifests here:
- The contract will execute the `TRSRY.setDebt` function with an amount of `15,000- 10,000 = 5,000 DAI` . This is incorrect because the debt should have been reduced by the entire `defundAmount.`
- This incorrect debt adjustment leads to a debt `misbalance` of `5,000 DAI` in the contract.
so this misbalance can disrupt accurate accounting of debt, and  leading to incorrect financial calculations and improper reward distributions.
## Impact
 the contract holding an incorrect amount of debt, which could disrupt the overall financial state and trustworthiness of the Clearinghouse improper reward distributions. 
## Code Snippet
- https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L275C5-L324C45
## Tool used

Manual Review

## Recommendation
Ensure that when defunding, the debt is correctly adjusted by subtracting the actual defund amount from the outstanding debt and  Use accurate accounting to prevent incorrect financial calculations.