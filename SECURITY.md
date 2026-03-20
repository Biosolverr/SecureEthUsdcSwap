## Supported Versions

| Version | Status |
|---|---|
| v9.0 FINAL | ✅ Supported (Current) |
| v8.x and below | ❌ Deprecated — known vulnerabilities |

---

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Please report security issues by emailing:  
**security@secureswap.example**

Include in your report:
- Contract version and deployment address (if applicable)
- Vulnerability description and attack vector
- Proof-of-concept or reproduction steps
- Estimated impact (funds at risk, affected users)

**Response SLA:**
- Acknowledgement within **48 hours**
- Detailed response within **7 days**

---

## Threat Model

This contract is designed to protect against:
- **MEV/Front-Running**: Commit-reveal pattern hides swap parameters until finalization
- **Griefing**: Collateral locks and reputation penalties discourage abuse
- **Rug Pulls**: Atomic execution ensures either both parties get their assets or neither does
- **Cross-Chain Timing Exploits**: Both parties can refund independently after reveal deadline
- **USDC Blacklisting**: Separated withdrawal functions prevent one token blocking the other

---

## Security Architecture

### Access Control

All state-changing functions are access-controlled at the function level. There is **no owner, admin, or upgradeable proxy** — the contract is fully immutable after deployment.

| Function | Permitted Callers |
|---|---|
| `commitInitiate()` | Anyone |
| `initiateFromCommit()` | Commitment creator only |
| `fund()` | Designated counterparty or any eligible address (open swap) |
| `complete()` | Initiator only |
| `claimAsCounterparty()` | Counterparty only |
| `refund()` — INITIATED state | Initiator only, after deadline + 15s |
| `refund()` — FUNDED state | Initiator OR counterparty, after revealDeadline + 15s |
| `cancel()` | Initiator only, before deadline + 15s |
| `withdrawEth()` | Caller withdraws own balance only |
| `withdrawUsdc()` | Caller withdraws own balance only |

---

### Reentrancy Protection

All external-call functions are protected by OpenZeppelin `ReentrancyGuard`. The Checks-Effects-Interactions pattern is strictly followed:

- State variables updated **before** any external call
- `pendingEthWithdrawals` / `pendingUsdcWithdrawals` deleted before transfer
- `_burnEth()` increments `burnedEthTotal` before the `call` to `0xdEaD`
- No external calls in `_executeCompletion()` — all payouts are queued

---

### Pull-Payment Pattern

Funds are never pushed to users. All payouts are queued in `pendingEthWithdrawals` and `pendingUsdcWithdrawals`. Users claim via `withdrawEth()` and `withdrawUsdc()` independently.

**ETH and USDC withdrawals are fully isolated.**  
A USDC blacklisting event cannot block ETH recovery. `withdrawBoth()` is intentionally absent to preserve this guarantee.

---

### MEV & Front-Running Protection

Swap parameters are hidden in a commitment hash submitted one transaction before `initiateFromCommit()`. The commitment binds:

```solidity
keccak256(abi.encode(
  secretHash, nonce, counterparty,
  ethAmount, usdcAmount, duration,
  msg.sender,           // sender bound into hash
  salt
))
```

The `msg.sender` binding prevents an attacker from hijacking a commitment they observed in the mempool.

---

### Timing Security

#### Timestamp Drift Buffer
A `TIMESTAMP_DRIFT_BUFFER` of 15 seconds is applied to all deadline checks to tolerate validator timestamp manipulation within the protocol-allowed range.

#### Window Adjacency (No Dead Zones)

complete() available:  [funded,    revealDeadline + 15s)   -- exclusive upper bound
refund()   available:  [revealDeadline + 15s, ∞)           -- FUNDED swaps
cancel()   available:  [initiated, deadline + 15s)         -- exclusive upper bound
refund()   available:  [deadline + 15s, ∞)                 -- INITIATED swaps

All windows are contiguous. There is **no period where a swap is frozen** with no available action for either party.

- `complete()` and `refund()` (FUNDED) share the boundary `revealDeadline + 15s`
- `cancel()` and `refund()` (INITIATED) share the boundary `deadline + 15s`

---

### Collusion Detection

Grief events between a pair are tracked using a canonical key:

```solidity
(lo, hi) = a < b ? (a, b) : (b, a);
pairGriefCount[lo][hi]++;
```

This ensures A→B grief and B→A grief both increment the same counter, enabling true bidirectional collusion detection without inflated mirrored values.

After `COLLUSION_THRESHOLD` (2) grief events, both addresses are flagged for `COLLUSION_FLAG_TIMEOUT` (30 days) and blocked from creating or funding swaps with each other.

---

### Secret Hash Replay Protection

`usedSecretHashes[secretHash]` is set to `true` permanently when a swap is initiated. It is **never cleared** on cancel or refund. This prevents a cancelled swap's secret hash from being reused in a new swap, closing a potential cross-chain replay vector.

---

### ETH Burn Integrity

ETH collateral burned during grief or cancellation is physically sent to `0xdEaD` via low-level `call`:

```solidity
(bool ok,) = payable(BURN_ADDRESS).call{value: amount}("");
if (!ok) revert EthBurnFailed();
```

**Assumption:** `0xdEaD` must be a pure EOA (no deployed code) on the target network. Verify this before deploying on L2s or non-mainnet EVM chains.

USDC collateral is burned logically (counter only) to avoid the risk of `0xdEaD` being USDC-blacklisted, which would permanently brick `refund()`.

---

### Rate Limiting

Per-address controls to limit spam and griefing surface:

| Limit | Value |
|---|---|
| Max concurrent active swaps | 5 per address |
| Initiate cooldown | 2 minutes between initiations |
| Fund cooldown | 1 minute between funding operations |
| Commit expiry | 10 minutes to use a commitment |

---

## Known Risks

### 1. BURN_ADDRESS Assumption
**Risk**: On L2s/sidechains, `0xdEaD` may be a contract with code, allowing someone to call it or drain its balance.

**Mitigation**:
- Verify `0xdEaD` is a pure EOA before deploying on non-mainnet networks
- Consider using an alternative burn address if needed
- Burned ETH is mathematically accounted for in `burnedEthTotal`

**Status**: ⚠️ **MEDIUM** (deployment-time check required)

### 2. USDC Token Assumptions
**Risk**: Non-standard ERC-20 implementations may break the contract.

**Mitigation**:
- This contract is designed specifically for USDC (Circle's implementation)
- Do not deploy with other tokens without thorough auditing
- SafeERC20 wrapper handles most non-standard cases

**Status**: ✅ **MITIGATED** (token-specific design)

### 3. Collusion Flag Expiry
**Risk**: After 30 days, a collusion flag expires even if both parties continue griefing.

**Mitigation**:
- Grief counter (`pairGriefCount`) never resets
- Any new griefing immediately re-triggers the flag
- 30-day timeout allows bad relationships to recover

**Status**: ✅ **BY DESIGN** (reputational rehabilitation)

### 4. Counterparty May Be Blacklisted by USDC
**Risk**: If counterparty is later blacklisted by USDC, they cannot withdraw their portion.

**Mitigation**:
- This is a USDC/Circle risk, not a contract design flaw
- Initiators should verify counterparty reputation before committing
- Separated withdrawals mean initiators can still withdraw ETH

**Status**: ✅ **UNAVOIDABLE** (external token risk)

### 5. Timing Window Precision
**Risk**: Validators can manipulate `block.timestamp` within ±15 seconds.

**Mitigation**:
- `TIMESTAMP_DRIFT_BUFFER = 15 seconds` accounts for this variance
- All window boundaries use the buffer consistently

**Status**: ✅ **MITIGATED** (buffer-based tolerance)

---

## Known Limitations

**1. Rate limits are per-address, not per-identity.**  
A single actor controlling multiple wallets can bypass per-address rate limits. This is a known trade-off of on-chain rate limiting.

**2. `0xdEaD` assumption on L2s.**  
If a contract is deployed at `0xdEaD` on the target network, `_burnEth()` may revert, blocking `refund()` and `cancel()`. Verify before deployment.

**3. USDC upgrade risk.**  
The contract depends on USDC's ERC-20 interface. A USDC contract upgrade that changes transfer semantics could affect `fund()`, `withdrawUsdc()`, or the logical burn accounting.

**4. Open swap MEV.**  
For open swaps (`counterparty == address(0)`), any eligible address can call `fund()`. An MEV searcher can front-run an intended counterparty. **For sensitive swaps, always specify a designated counterparty.**

**5. `usedSecretHashes` storage growth.**  
The mapping grows monotonically and is never pruned. This is intentional for replay protection. Each new swap requires a fresh `secret` and `nonce`.

---

## Attack Vectors & Mitigations

| Attack | Vector | Mitigation |
|---|---|---|
| **Grief + Collusion Bypass** | A and B grief once, then stop | Grief count is permanent; flag re-triggers on any new grief |
| **Commit Reuse** | Attacker replays someone else's commitment | Commitment includes `msg.sender`; only creator can reveal |
| **Secret Brute-Force** | Guess secret with low entropy | `secretHash = keccak256(abi.encode(secret, nonce))`; must guess both |
| **Open Swap Front-Running** | MEV-bot funds before intended counterparty | First funder becomes counterparty; others get `NotCounterparty` |
| **Rate Limit Bypass** | Use multiple wallets | Per-address design; no cross-wallet limiting (known limitation) |
| **Collusion + Designated Counterparty** | Flag exists but counterparty can still fund | `fund()` checks collusion flag for designated counterparties |

---

## Safe Practices for Users

### Initiators
1. **Generate strong random secrets**: Use cryptographically secure randomness
2. **Verify counterparty reputation**: Check `getReputation(counterparty)` before initiating
3. **Set reasonable durations**: 1-3 hours is typical; avoid very long timeouts
4. **Keep secret safe**: Do not share or log it until `complete()` is called
5. **Monitor reveal window**: Complete the swap well before `revealDeadline + 15s`

### Counterparties
1. **Verify initiator reputation**: Check their completed trades and griefed count
2. **Confirm swap terms**: Verify amounts match what you expect
3. **Fund promptly**: Don't wait until the last moment; fund well before `revealDeadline`
4. **Be ready to withdraw**: Monitor the contract after swap is complete
5. **Watch for collusion flags**: If flagged, you cannot participate in new swaps with that initiator for 30 days

---

## Audit Status

| Auditor | Version | Date | Report |
|---|---|---|---|
| — | — | — | Not yet audited |

> ⚠️ **This contract has not been independently audited. Use in production at your own risk.**

---

## Bug Bounty

No formal bug bounty program is currently active. Responsible disclosure is appreciated and will be credited.
