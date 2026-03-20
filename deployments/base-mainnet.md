# Base Mainnet Deployment

This document records the canonical deployment of **SecureEthUsdcSwap** on Base Mainnet.

All parameters below correspond exactly to the verified on-chain contract and are provided for auditability, reproducibility, and integration reference.

---

## Network

- Network: Base Mainnet
- Chain ID: 8453

---

## Contract

- Name: SecureEthUsdcSwap
- Version: v9
- Address:  
  0x19F148f34c10Cdb69b2356EFE223296be653092b

---

## Deployment Transaction

- Transaction hash:  
  0xc0f330c67a168acc6d05258be6757adb64447a4656ef429e2c4d0788ad3b5abe
- Block number:  
  43601082

---

## Constructor Parameters

The contract was deployed with the following constructor argument:

- USDC (Base Mainnet):  
  0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

No other constructor parameters are used.

---

## Verification

- Status: Verified on BaseScan
- Verification method: Solidity Standard JSON Input
- Compiler version: 0.8.28
- Optimization: Enabled (200 runs)
- viaIR: true
- EVM version: cancun

The verified source code and the deployed bytecode are identical.

---

## Notes

- This deployment represents the canonical production instance of SecureEthUsdcSwap.
- The contract is non-upgradeable and has no admin or owner privileges.
- All protocol rules are enforced exclusively by the deployed bytecode.
