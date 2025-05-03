# Marginfi-v2 Independent Audit Report

## Executive Summary
I conducted a comprehensive security, performance, and best-practices audit of the open-source Solana program **marginfi-v2** (GitHub: https://github.com/mrgnlabs/marginfi-v2). I did not find any critical or high-severity issues. The audit revealed **2 Medium-severity** items related to oracle handling and administrative controls, **6 Low-severity** best-practice observations, and **5 Informational** notes. Overall, marginfi-v2 is well-designed and secure, employing strong access controls, safe math, and robust risk checks. My recommendations focus on enhancing oracle safety, refining fee handling, and improving governance and documentation.

## Vulnerability Classifications

| Severity        | Description                                                                          |
|-----------------|--------------------------------------------------------------------------------------|
| **Critical**    | Issues that could lead to catastrophic loss of funds or data exposure.              |
| **High**        | Significant vulnerabilities that could result in major financial loss if exploited. |
| **Medium**      | Potential risks to protocol integrity or user funds under certain conditions.       |
| **Low**         | Minor issues or best-practice gaps with limited exploitability.                     |
| **Informational** | Helpful observations and suggestions that improve robustness but pose no immediate risk. |

## Scope
- **Repository:** https://github.com/mrgnlabs/marginfi-v2  
- **Language:** Rust with the Anchor framework  
- **Network:** Solana Mainnet-Beta  
- **Audit Period:** April 2025  
- **Coverage:** All on-chain instruction handlers, the risk engine, flashloan logic, and administrative functions.

## Summary of Findings

| ID      | Area                 | Description                                                              | Severity       |
|---------|----------------------|--------------------------------------------------------------------------|----------------|
| MED-01  | Oracle Handling      | Lack of staleness and confidence checks on price oracles.                | Medium         |
| MED-02  | Admin Governance     | Admin can change critical parameters and withdraw fees without delay.    | Medium         |
| LOW-01  | Assertion Usage      | Use of `assert!()` could panic rather than return a controlled error.    | Low            |
| LOW-02  | Fee Dust             | Fractional origination fees are truncated, allowing small fee avoidance. | Low            |
| LOW-03  | Disabled Repay       | Disabled accounts cannot repay to recover, even though repay improves safety. | Low        |
| LOW-04  | Event Timing         | Events emitted before final health checks may appear on failed txns.     | Low            |
| LOW-05  | Flashloan Docs       | Flashloan flow not clearly documented in IDL or events.                  | Low            |
| LOW-06  | Panic vs Error       | Sanity checks use `assert!` instead of `check!` for safe error codes.    | Low            |
| INF-01  | Documentation        | Recommend adding NatSpec comments for clarity.                           | Informational  |
| INF-02  | UX Behavior          | Clarify `withdraw_all` and `repay_all` semantics for end users.          | Informational  |
| INF-03  | Health Cache         | Explain how `health_cache` is used and persisted.                        | Informational  |
| INF-04  | Compute Profiling    | Suggest on-chain profiling of compute unit usage.                        | Informational  |
| INF-05  | Governance Tools     | Recommend multi-sig/timelocks for critical admin actions.                | Informational  |

## Detailed Findings

### Oracle Handling (MED-01)
**Location:** RiskEngine invoked during borrow, withdraw, liquidation, and flashloan end.  
**Observation:** No checks on oracle update timestamps or confidence intervals, leaving risk of stale or manipulated price data.  
**Impact:** Medium — attackers could exploit oracle weaknesses to borrow extra funds or evade liquidation.  
**Recommendation:**  
- Add staleness checks for oracle feeds (e.g. reject if last update > threshold).  
- Factor in confidence intervals (e.g. use `price ± confidence`).  
- Consider multi-oracle aggregation or median prices.

### Admin Governance (MED-02)
**Location:** Admin-only instructions (bank creation, parameter updates, fee withdrawals).  
**Observation:** Single admin key can make immediate parameter changes or withdraw fees with no delay or multisig.  
**Impact:** Medium — a compromised admin key could alter parameters maliciously or drain fees.  
**Recommendation:**  
- Implement timelocks (e.g. 24–48 h) on critical config changes.  
- Migrate to a multi-sig or on-chain governance program as the admin authority.  
- Emit detailed config-change events with old/new values.

### Assertion Usage (LOW-01)
**Location:** Liquidation fee sanity check (`assert!(insurance_fund_fee >= 0)`).  
**Observation:** Assertion panics on failure, consuming all compute units instead of returning a controlled error.  
**Impact:** Low — unlikely in production but would waste compute and complicate debugging.  
**Recommendation:**  
- Replace `assert!` with `check!(insurance_fund_fee >= 0, MarginfiError::MathError)`.

### Fee Dust (LOW-02)
**Location:** Origination fee calculation (I80F48 → u64 truncation).  
**Observation:** Fractional fee parts <1 token unit are dropped, enabling small-scale fee avoidance.  
**Impact:** Low — limited economic risk but worth addressing for complete fee capture.  
**Recommendation:**  
- Accumulate fractional fees in I80F48 until ≥1 unit, then convert.  
- Or enforce a minimum fee of 1 unit when any fee >0 is due.

### Disabled Repay (LOW-03)
**Location:** `lending_account_repay` blocks when account `DISABLED_FLAG` is set.  
**Observation:** Users cannot repay to reduce debt if their account is disabled, even though repayment improves safety.  
**Impact:** Low — prevents self-remediation of under-collateralized positions.  
**Recommendation:**  
- Allow repay on disabled accounts, or introduce a dedicated “recovery” flag/instruction.

### Event Timing (LOW-04)
**Location:** Withdraw and borrow handlers emit events before final health validation.  
**Observation:** Events may appear in logs even when transactions revert on health failure.  
**Impact:** Low — log noise can mislead off-chain indexing.  
**Recommendation:**  
- Emit events **after** successful health checks and state commits.

### Flashloan Docs (LOW-05)
**Location:** Flashloan start/end flow not clearly described in IDL or on-chain events.  
**Observation:** Integrators may misorder instructions or omit required oracle accounts, causing silent failures.  
**Impact:** Low — developer-experience issue.  
**Recommendation:**  
- Annotate IDL and NatSpec with the required sequence and expected accounts.  
- Emit `FlashloanStarted(end_ix_index)` and `FlashloanEnded` events.

### Panic vs Error (LOW-06)
**Location:** Internal sanity checks use `assert!` instead of `check!`.  
**Observation:** Panics are less transparent than structured error returns and consume full compute budget on failure.  
**Impact:** Low — affects error clarity and compute usage.  
**Recommendation:**  
- Convert all `assert!` checks into `check!` with specific `MarginfiError` variants.

## Recommendations

1. **Harden Oracle Feeds** with staleness and confidence checks; consider multi-oracle aggregation.  
2. **Strengthen Admin Governance** via timelocks, multi-sig, and transparent config-change events.  
3. **Improve Fee Handling** by accumulating fractional “dust” fees and enforcing minimum fees.  
4. **Refine Error Handling**: replace all panics (`assert!`) with controlled `check!` errors.  
5. **Enhance Documentation**: add NatSpec comments, clarify flashloan flow, and UX notes for `withdraw_all`/`repay_all`.  
6. **Clarify Event Emission**: only emit events after successful health checks to avoid misleading logs.  
7. **Allow Repay on Disabled Accounts** or provide a dedicated recovery mechanism.  
8. **Profile Compute Usage** regularly and finalize governance authority (e.g., remove upgrade key) once stable.

## Overall Assessment
I conclude that **marginfi-v2** is robustly secured, with strong access controls, safe CPI patterns, and comprehensive risk validations. Addressing the medium-severity items on oracle safety and governance, along with the low-severity best-practice enhancements, will further solidify the protocol’s reliability and trustworthiness.
