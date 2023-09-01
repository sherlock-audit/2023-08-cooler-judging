Special Mauve Pangolin

medium

# Withdraw tokens to treasury in `_onRepay` in ClearingHouse if emergency shutdown
## Summary

In the event of an emergency shutdown, any time a loan is repaid to the Clearing House afterwards, the funds should be sent to the treasury. This however is currently not done -- the funds just sit in the Clearing House contract until `rebalance` is called, which could take `FUND_CADENCE` time. 

## Vulnerability Detail

If there is some type of exploit in the protocol, leading to an emergency shutdown, then loans that are repaid afterwards will sit in the ClearingHouse contract because `_onRepay` does not withdraw the funds to the treasury in the event of an emergency shutdown:

```solidity
    function _onRepay(uint256, uint256 amount_) internal override {
        _sweepIntoDSR(amount_);

        // Decrement loan receivables.
        receivables = (receivables > amount_) ? receivables - amount_ : 0;
    }
```

However, correct behavior is for the funds to be immediately be sent to the treasury. Currently, `rebalance()` will potentially send the funds to the treasury once it is called, but you have to potentially wait `FUND_CADENCE` time for this to happen (and also remember calling it is not immediate in the first place, so someone could drain the funds in the time it takes for it to be called). 

## Impact

Repaid loans will sit in the ClearingHouse contract despite an emergency shutdown (which may lead to them being drained in the event of an exploit), when they should instead be sent to the treasury

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L252-L257

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L281

## Tool used

Manual Review

## Recommendation
Manually check `active` at the end of `_onRepay` and if false, send funds to the treasury. Furthermore, I would recommend `rebalance()` to ignore the `FUND_CADENCE` time check in the event `active` is false. 