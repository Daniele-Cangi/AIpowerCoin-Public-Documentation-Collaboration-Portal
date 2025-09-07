# AIpowerCoin — Technical Memo & Narrative Overview

## 1. Why it exists

European energy markets fluctuate: price volatility, fragmented strategic reserves, inefficiency in capital management.

"Classic" cryptocurrencies are unstable because they have no direct connection to real assets.

AIpowerCoin was born to solve both problems: **a token anchored to real energy consumption**, with a coverage mechanism that transforms available energy into stable digital value.

---

## 2. The system in summary

Every day:

1. **EU energy data** — the script collects consumption volumes from ENTSO-E (or from simulated datasets).

2. **Commit ratio (α)** — a prudential quota (≈18%) of total energy is committed as "coverage".

3. **Coverage planner** — calculates the market value of this coverage, compares with token supply in circulation and determines stability.

4. **Attestation** — results are cryptographically signed by multiple independent roles (data, risk, ops) → guarantee of integrity and distributed consensus.

5. **Dashboard** — users and operators see transparency: energy stock, implicit value per token, coverage ratio.

---

## 3. Benefits

- **Stability** — the token is not disconnected from reality: value derives from a daily energy base, with coverage ratio (CR) always >1.

- **Transparency** — every day a `attestation_YYYY-MM-DD.json` file signed and publicly verifiable.

- **Resilience** — the commit ratio limits risk: not all energy volume is allocated, reserves and margins are preserved.

- **Controlled scalability** — the planner calculates how many new units can be issued without compromising coverage (mint_suggested).

- **Auditability** — anyone can verify data with open source scripts and compare signed digests.

---

## 4. How it addresses the reserves problem

Energy markets suffer from two recurring problems:

1. **Insufficient reserves during demand peaks.**

2. **Excess immobilized capital during low volatility periods.**

AIpowerCoin adopts an **elastic mechanism**:

- Commit ratio (α) → commits only a fraction of energy (e.g. 18%), so the system preserves safety buffers.

- Coverage ratio (CR) → measures how much coverage exists relative to issued tokens; if it drops below threshold, new token creation stops.

- Daily mint cap → even when CR is high, daily growth is limited (e.g. 5% maximum) to avoid overshooting.

This means that reserves **are never exhausted** to sustain monetary supply: system stability has priority over growth.

---

## 5. How it generates value stably

- Every Wh of "committed" energy is converted to economic value (`€/MWh` from market index).

- This total value divided by tokens in circulation determines the **implicit value per token**.

- If the implicit value is much > target (e.g. 97 €/token vs 1.50 € target), the system is **hyper-covered** → great stability and future margins.

- If the implicit value approaches the target, the planner reduces emissions until zeroing them → no collapse risk.

This way AIpowerCoin functions as an **algorithmic energy bank**:  
locks real capital (energy), releases digital fractions, maintains balance between stability and growth.

---

## 6. Vision

AIpowerCoin unites:

- **European energy networks** (real ENTSO-E data)

- **Modern cryptography** (Ed25519 signatures, multi-role attestations)

- **Blockchain-like transparency** (public digests, verifiable JSONs)

- **Systemic prudence** (commit ratio, mint cap)

The result is a system that **does not promise speculative returns**, but **stability and auditability**:  
a token that represents **computational and energy value supported by real data**.

---

## 7. Quick glossary

- **CR (Coverage Ratio)** = (€/token implicit) ÷ (€/token target).  
  > CR ≥ 1.1 → issuance allowed; CR < 1.1 → stop mint.

- **Commit ratio (α)** = quota of energy "locked" as guarantee (typically 0.18).

- **Attestation** = daily file with digest, values and multi-role signatures.

- **Digest** = canonical hash of attestation (immutable, verifiable by anyone).