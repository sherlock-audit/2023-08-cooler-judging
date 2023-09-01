Recumbent Syrup Cod

high

# Missing Check For Maching Collateral and Debt tokens in claimDefaulted
## Summary
Missing check if tokens for collateral and debt in cooler is same (DAI and gOHN) in claimDefaulted function in Clearinghouse contract. Which can be used by attacker to exploit reward for keeper. 
## Vulnerability Detail
Attacker can create cooler with two FakeTokens (create by himself) lets call them FakeToken1 and FakeToken2. So after attacker create cooler with them he can make request to borrow 5000 FakeTokens1 for 1000 FakeTokens2 as collateral. After loan expire he can call claimDefaulted function in Clearinghouse contract and since there no have check if tokens are DAI and gOHM function will pass , Clearinghouse will receive 1000 FakeTokens2 and as reward function will send him gOHM tokens as you can see here
```javascript
// Reward keeper.
        gOHM.transfer(msg.sender, keeperRewards);
```
## Impact
Attacker can get free gOHM tokens
## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L191-L245
## Tool used

Manual Review

## Recommendation
To mitingate this issue adding check in function for token mismatch
```javascript
 if (coolers[i].collateral() != gOHM || cooler[i].debt() != dai)
            revert BadEscrow();
```