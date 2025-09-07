# AIpowerCoin — Technical Whitepaper v0.2

## 0. Executive Summary
AIpowerCoin connects European energy data with a stable digital token.  
Every day the system:
- collects electrical consumption (ENTSO-E),
- calculates an energy reserve quota (commit ratio, α≈0.18),
- evaluates available coverage and stability ratio (Coverage Ratio, CR),
- issues a cryptographically signed attestation,
- updates a transparent dashboard.

**New features (v0.2)**:
- Attestations with `digest_signed` (SHA-256 on payload pre-signatures) and **Ed25519 multisig 2-of-3** signatures (roles: data, risk, ops).
- Coverage band: **CR ∈ [1.10, 1.20]**.
- Daily mint cap (e.g. 5% of S).
- Automatic freeze if CR < 1.10.
- Operational runbook with daily automation (Windows Task Scheduler).

---

## 1. Context
The European energy market presents:
- price volatility,
- difficulties in reserve management,
- need for stable and transparent instruments.

Traditional cryptocurrencies are unstable or disconnected from real assets.  
AIpowerCoin combines energy data and prudent monetary rules to build a resilient and verifiable token.

---

## 2. Conceptual architecture
- **Energy as base**: every Wh consumed is tracked, a fraction α is committed.
- **Digital value**: the committed reserve translates to stock value.
- **Coverage (CR)**: measures token/reserve stability.
- **Attestation**: daily JSON file, signed and verifiable.
- **Dashboard**: interface for users and operators.

---

## 3. Monetary policy
- **Safety band**: maintain **CR ∈ [1.10, 1.20]**.
- **Mint enabled**: if **CR ≥ 1.10**, with **daily cap** (e.g. 5% of S).
- **Freeze**: if **CR < 1.10**, issuance blocked until recovery.
- **Target supply**:
  \[
  S_\text{target} = \frac{V_\text{reserve}}{P_T \cdot \text{CR}_\min}
  \]
  with \(\text{mint\_suggested} = \max(0, \min(S_\text{target} - S, \text{daily\_cap}))\).
- **Utility redeem**: services at target \(P_T\) (not pro-rata reserve) → arbitrage prevention.

---

## 4. Attestation process
### 4.1 Structure
Each day generates `attestation_YYYY-MM-DD.json` with:
- committed energy data,
- CR calculations, S, suggested mint,
- meta (version, algorithms, digest, signatures).

### 4.2 Canonical digest
- Pre-signature payload with `meta.signers=[]`, `meta.signatures=[]`.
- SHA-256 on canonical JSON (`sort_keys=true`, compact separators).
- Inserted in `meta.digest_signed`.

### 4.3 Signature v2 (multisig 2-of-3)
Signed message (domain separated):
Roles: `data`, `risk`, `ops`.  
Ed25519 signatures collected in `meta.signatures`.  
Public keys in `meta.signers`.

### 4.4 Verification
The verifier:
1. reconstructs the canonical digest,
2. recreates the message,
3. validates ≥2 consistent signatures.  
Result: `signatures_ok = N, required = 2`.

---

## 5. Technical architecture
### 5.1 Modules
- `update_grid_eu_strict.py` → ENTSO-E data loader per zone.
- `coverage_planner.py` → CR calculation, S_target, mint.
- `sign_attestation.py` → adds digest, signatures.
- `verify_attestation.py` → signature validation.
- `dashboard.py` → web interface.

### 5.2 Operational runbook
1. **Loader**: saves `grid_by_zone`, `grid_summary`, updates `grid_load.csv`.
2. **Planner**: generates `attestation_YYYY-MM-DD.json`.
3. **Signature**: multisig 2-of-3.
4. **Verification**: `OK` if ≥2 valid signatures.
5. **Dashboard**: shows latest data.

### 5.3 Automation
- Daily task (00:20): loader.
- Daily task (00:25): planner → signer (≥2 roles) → verifier.
- Separate logs (`logs/*`).
- Minimum zone coverage control.

---

## 6. Benefits
- **Stability**: daily energy anchoring.
- **Transparency**: signed and verifiable JSON attestations.
- **Resilience**: buffer through commit ratio.
- **Controlled scalability**: emissions limited by CR and cap.
- **Auditability**: open source and public digests.

---

## 7. Governance and transparency
**Public (repo):**
- economic model,
- file formats,
- verification guide,
- signer public keys,
- attestation examples.

**Private/local:**
- ENTSO-E tokens,
- private keys,
- production jobs,
- complete logs,
- raw datasets.  
`data/` and `logs/` excluded via `.gitignore`.

---

## 8. Glossary
- **CR (Coverage Ratio)** = implicit value/token ÷ target.  
- **Commit ratio (α)** = committed energy quota (~0.18).  
- **Attestation** = signed daily file.  
- **Digest** = canonical hash pre-signatures.  

---

## 9. Appendices
### 9.1 File formats
- `grid_load.csv` → `date,grid_wh_dc,grid_wh_dc_commit`
- `grid_by_zone_YYYY-MM-DD.csv` → `name,eic,wh`
- `grid_summary_YYYY-MM-DD.json` → `{day,zones_ok,wh_dc_raw,wh_dc_commit,alpha_base,alpha_effective,zones:[...]}` 
- `attestation_YYYY-MM-DD.json` → [detailed structure continues...]