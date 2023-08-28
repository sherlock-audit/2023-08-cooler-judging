Breezy Myrtle Anteater

high

# `emergency_shutdown` role is not enough for emergency shutdown.
## Summary

There are two protocol roles, `emergency_shutdown` and `cooler_overseer`. The `emergency_shutdown` should have the ability to shutdown the Clearinghouse.

However, in the current contract, `emergency_shutdown` role does not have said ability. An address will need both `emergency_shutdown` and `cooler_overseer` to perform said action.

We have also confirmed with the protocol team that the two roles will be held by two different multisigs, with the shutdown multisig having a lower threshold and more holders. Thereby governance will not be able to act as quickly to emergencies than expected.

## Vulnerability Detail

Let's examine the function `emergencyShutdown()`:

```solidity 
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

This has the modifier `onlyRole("emergency_shutdown")`. However, this also calls function `defund()`, which has the modifier `onlyRole("cooler_overseer")`

```solidity
function defund(ERC20 token_, uint256 amount_) public onlyRole("cooler_overseer") {
```

Therefore, the role `emergency_shutdown` will not have the ability to shutdown the protocol, unless it also has the overseer role.

### Proof of concept

To get a coded PoC, make the following modifications to the test case:
- In `Clearinghouse.t.sol`, comment out line 125 (so that `overseer` only has `emergency_shutdown` role)
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/test/Clearinghouse.t.sol#L125

```solidity
//rolesAdmin.grantRole("cooler_overseer", overseer);
rolesAdmin.grantRole("emergency_shutdown", overseer);
```

- Run the following test command (to just run a single test `test_emergencyShutdown()`):
```sh
forge test --match-test test_emergencyShutdown
```

The test will fail with the `ROLES_RequireRole()` error.
## Impact

`emergency_shutdown` role cannot emergency shutdown the protocol

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L339
https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L360-L372

## Tool used

Manual Review, Foundry/Forge

## Recommendation

There are two ways to mitigate this issue:
- Separate the logic for emergency shutdown and defunding. i.e. do not defund when emergency shutdown, but rather defund separately after shutdown. 
- Move the defunding logic to a separate internal function, so that emergency shutdown function can directly call defunding without going through a modifier.
