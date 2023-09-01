Immense Syrup Shetland

high

# Clearinghouse.sol#claimDefaulted() - `Clearinghouse` doesn't approve the `MINTR` to handle tokens in his name, which bricks the entire function.
## Summary
`Clearinghouse` doesn't approve the `MINTR` to handle tokens in his name, which bricks the entire function.

## Vulnerability Detail
Inside `claimDefaulted` on the [last line](https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Clearinghouse.sol#L244) we call `MINTR.burnOhm` which in turn calls [OHM.burnFrom](https://github.com/OlympusDAO/olympus-v3/blob/19236eb1c02464df8fb79c7b59b7195d7511b338/src/modules/MINTR/OlympusMinter.sol#L50-L61). The [docs for MINTR.burnFrom](https://docs.olympusdao.finance/main/technical/contract-docs/modules/MINTR/OlympusMinter/#burnohm) state: "Burn OHM from an address. Must have approval.". We can confirm that this is the case when looking at `OHM` source code and it's `burnFrom`. I found 2 `OHM` tokens that are currently deployed on mainnet, so I'm linking both their addresses: https://etherscan.io/token/0x383518188c0c6d7730d91b2c03a03c837814a899#code, https://etherscan.io/token/0x64aa3364f17a4d01c6f1751fd97c2bd3d7e7f1d5#code. Both addresses use the same `burnFrom` logic and in both cases they require an `allowance`. Nowhere in the contract do we approve the `MINTR` to handle `OHM` tokens in the name of `Clearinghouse`, in fact `OHM` isn't even specified in `Clearinghouse`.  

Side note:
The test `testFuzz_claimDefaulted` succeeds, because `MockOhm` is written incorrectly. When `burnFrom` gets called `MockOhm` calls the inherited `_burn` function, which burns tokens from `msg.sender`. The mock doesn't represent how the real `OHM.burnFrom` works.
## Impact
`Claimdefault` will always revert.

## Code Snippet
https://github.com/sherlock-audit/2023-08-cooler/blob/6d34cd12a2a15d2c92307d44782d6eae1474ab25/Cooler/src/Clearinghouse.sol#L244
## Tool used
Manual Review

## Recommendation
Add a variable `ohm` which will be the `OHM` address and approve the necessary tokens to the `MINTR` before calling `MINTR.burnOhm`.
