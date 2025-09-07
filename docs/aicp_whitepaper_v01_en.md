# AIpowerCoin (AICP) — Whitepaper v0.1 (Draft)

> **Status:** draft for internal review.
> **Objective:** describe the rationale, economic and technical design, monetary policy rules and attestation process for AICP.

---

## 0. Executive Summary
AIpowerCoin (AICP) is a unit of account anchored to **physical resources** (electrical energy and computing capacity). The operational value of the token is stabilized around a **price target** \(P_T\;=\;1{,}50\,€ / \text{AICP}\) through a **reserve** measured daily (contractable energy + computing assets) and a **supply policy** that is simple and verifiable.

- **Transparent collateral:** energy aggregated from ENTSO-E (documentType=A65, processType=A16), calculated by **bidding zone** with PTU-aware resolution, plus computing assets valued at prudential market rates.
- **Public attestation:** each day a **attestation JSON** is published with \(\text{Wh}\), \(\alpha\), reserve value, **S** (supply), **CR** (coverage ratio) and **mint_suggested**.
- **Monetary policy:** **mint** is allowed when \(\text{CR} \ge 1{,}10\) and frozen when \(\text{CR} < 1{,}10\). Stability target: \(\text{CR}\) in band **[1.10, 1.20]**.
- **Utility redeem:** AICP is redeemable **in services** (compute/energy) **at the rate linked to the target** \(P_T\) — not at the pro-rata value of the reserve — preventing arbitrage.

---

## 1. Motivation and objectives
**Problem:** most crypto-assets lack material anchoring and suffer from excessive volatility.

**Objective:** define a utility-currency that measures and exchanges **computational work and energy** with **quantitative guarantees** and **simple rules**.

**Requirements:**
- **Public, reproducible** data (ENTSO-E for energy, verifiable inventories for compute).
- **Simple algorithms**, few parameters, auditable.
- **Operational resilience** (fallback, PTU management, time zones, per-zone datasets).

---

## 2. Data and methodology
### 2.1 Energy source
- **ENTSO-E Transparency Platform**: endpoint **A65 / A16** (load measurement). Query by **outBiddingZone_Domain** (EIC) and **local day** of the zone.
- **Timezone/DST-safe**: for each zone a **local TimeInterval** (00:00—24:00 local) is constructed and sent in UTC.

### 2.2 Energy calculation
For each zone \(z\) on day \(d\):
\[
\text{Wh}_{z,d} = \sum_{i \in \text{PTU}} \big( \text{MW}_{z,d,i} \times \text{PTUDuration}_{z,d,i}\;[\text{h}] \times 10^6 \big)
\]

Total energy and "committable" (prudential) with **commit ratio** \(\alpha \in (0,1)\):
\[
\text{Wh}_{d}^{\text{raw}} = \sum_z \text{Wh}_{z,d}, \qquad \text{Wh}_{d}^{\text{commit}} = \alpha \cdot \text{Wh}_{d}^{\text{raw}}
\]

Valuation at price \(P_E\) (€/MWh):
\[
V_{\text{energy},d} = \big(\text{Wh}_{d}^{\text{commit}}/10^6\big) \cdot P_E
\]

### 2.3 Computing assets
Prudential inventory (example): GPU **H100** off-peak valued at \(c_{\text{GPU}}\;€\) each. Value:
\[
V_{\text{compute}} = N_{\text{GPU}} \cdot c_{\text{GPU}}
\]

### 2.4 Reserve value and coverage
\[
V_{\text{reserve}} = V_{\text{energy}} + V_{\text{compute}}
\]

Effective supply: \(S = S_{\text{base}} \cdot \hat U\), with \(\hat U \in (0,1]\).  
**Coverage Ratio**:
\[
\text{CR} = \frac{V_{\text{reserve}}}{P_T \cdot S}
\]

**Supply target** to bring \(\text{CR}\) back to threshold \(\text{CR}_{\min}:\)
\[
S_{\text{target}} = \frac{V_{\text{reserve}}}{P_T \cdot \text{CR}_{\min}}\quad;\quad \text{mint\_suggested} = \min\big( S_{\text{target}}-S,\; \text{daily\_cap}\big)
\]

---

## 3. Monetary policy
- **Safety band**: \(\text{CR}\in[1.10,1.20]\).
- **Mint**: allowed if \(\text{CR}\ge1.10\), limited by **daily cap** (e.g., 5% of \(S\)).
- **Freeze**: if \(\text{CR}<1.10\) → no mint.
- **Defense**: if \(\text{CR}<1.00\) → priority to energy/asset purchases until \(\text{CR}\ge1.10\).
- **Governance parameters**: \(\alpha\), \(P_E\), \(P_T\), \(\text{CR}_{\min}\), **cap**, zone list.

**Note on redemption**: AICP redemption occurs in **services** at a rate consistent with \(P_T\); **not** a pro-rata right to monetized collateral.

---

## 4. Attestation process
### 4.1 Pipeline
1. **Loader**: `update_grid_eu_strict.py` → calculates \(\text{Wh}\) per zone (PTU-aware), sums, applies \(\alpha\), produces:
   - `grid_load.csv` (daily history: `date, grid_wh_dc, grid_wh_dc_commit`)
   - `grid_by_zone_YYYY-MM-DD.csv` (zone detail)
   - `grid_summary_YYYY-MM-DD.json` (day summary)
2. **Planner**: `coverage_planner.py` → reads `grid_load.csv`, adds computing asset value, calculates \(\text{CR}\), \(S_{\text{target}}\), `mint_suggested`, produces `attestation_YYYY-MM-DD.json`.
3. **Dashboard**: `dashboard.py` (local) → exposes `/api/latest` and HTML view.

### 4.2 Signature and oracle (v1 → v2)
- **v1 (off-chain)**: JSON hash (SHA-256) published with timestamp.
- **v2 (on-chain)**: digital signature (multisig key) and hash publication to Oracle contract.

---

## 5. Technical architecture
- **Zone set**: `data/zones.csv` (name, EIC, timezone) — includes Nordic splits (SE1-SE4, NO1-NO5) to avoid double counting.
- **ENTSO-E API**: uses only `outBiddingZone_Domain`. Fallback between local `TimeInterval` (UTC) and `periodStart/End` UTC.
- **PTU-aware**: respects `Resolution` (PT15M/PT30M/PT60M).
- **Resilience**: retry on 5xx/timeout, parsing ACK 400 for reasons (e.g., *Mandatory parameter*, *Invalid EIC*).

**Pseudocode (loader)**:
```
for zone in zones:
  xml = GET(A65/A16, outBiddingZone=zone, TimeInterval=local_day_UTC)
  if fail: xml = GET(A65/A16, outBiddingZone=zone, periodStart/End UTC)
  Wh_zone = sum(MW_i * PTU_h_i * 1e6)
sum_zones = Σ Wh_zone
commit = α * sum_zones
write CSV/JSON
```

**Pseudocode (planner)**:
```
read grid_load.csv (last day)
V_energy = (Wh_commit/1e6) * P_E
V_compute = units * €/unit
V_reserve = V_energy + V_compute
S = S_base * U_hat
CR = V_reserve / (P_T * S)
S_target = V_reserve / (P_T * CR_min)
mint_suggested = min(S_target - S, daily_cap)
write attestation.json
```

---

## 6. Economic analysis (summary)
- **Capital efficiency**: maximized by adjusting **S** to maintain \(\text{CR}\) in band.  
- **Robustness**: prudential \(\alpha\) absorbs partial absences (missing zones: IE/GB...), multiple fallbacks.
- **Risks**: \(P_E\) variation, feed unavailability, parsing error, ENTSO-E schema changes; mitigations: \(\text{zones\_ok}\) thresholds, retries, daily tests, dataset version.

---

## 7. Governance and parameters
- Mutable parameters (multisig): alpha, P_E, P_T, CR_min, daily_cap, zones.csv, compute inventory.
- Transparency: repository publication with scripts, checksums, daily datasets, signed attestations.

### 7.1 Transparency vs security: what we publish
**Always public**
- Economic model (CR formulas, alpha, mint/freeze policy), redeem rules (at target, not pro-rata), data sources (ENTSO-E A65/A16), PTU/DST methodology.
- Governance parameters and modification process (multisig, quorum).
- File formats and signed attestations (hash) with verification guide.

**In appendix/repo**
- zones.csv (EIC list and timezones) and examples of grid_summary_*.json / attestation_*.json.

**Not public (operational/security)**
- Credentials/tokens, private signing keys, infrastructure details, schedules/rate-limit limits, sensitive commercial contracts.

- **Mutable parameters** (multisig): \(\alpha, P_E, P_T, \text{CR}_{\min}, \text{daily\_cap}\), `zones.csv`, compute inventory.
- **Transparency**: repository publication with scripts, checksums, daily datasets, signed attestations.

---

## 8. Compliance & legal notes (not advice)
- AICP is utility token for service consumption; no return promise; financial uses subject to local regulation.
- Data and rules publication for independent audit.

## 8-bis. Attestation signing & Oracle v2
### v1 — Off-chain attestation signing
Objective: make daily attestation publicly verifiable that it has not been altered.

JSON canonicalization
- UTF-8 encoding without BOM.
- Sorted keys (sort_keys=true), compact separators ("," and ":"), no superfluous whitespace.
- Numbers in standard decimal format; dates as YYYY-MM-DD.

Digest
- hash_algo = sha256 on canonicalized JSON -> digest = 0x...

Signature
- sig_algo = ed25519 (alternative: secp256k1/ECDSA).
- Multisig 2-of-3: signers = {data, risk, ops}. Publish public keys in repo.
- Signed message (domain separated):
  "AICP|day=<YYYY-MM-DD>|digest=<0x..>|v=1|chain=EVM:<id>".
- Published payload: attestation JSON + meta {version, hash_algo, sig_algo, domain, chain_hint, signers(pubkey), signatures(signer, sig, ts)}.

Verification (steps)
1) Reconstruct canonical JSON of attestation, calculate SHA-256 and compare digest.
2) Verify at least 2 valid signatures on message (known public keys).
3) Check that day matches and version/chain_hint are expected.

### v2 — On-chain Oracle (EVM-compatible)
AICPOracleV2 Contract
- postAttestation(day, digest, fields, signatures) -> verifies 2-of-3 signatures, registers digest and minimal fields (CR, S, alpha, eurPerMwh, version).
- Events: AttestationPosted(day, digest, CR, S, alpha, eurPerMwh, version); clients index via events.
- Minimal storage: only latest digest per day (reduces gas); details remain off-chain in signed JSONs.

Security
- Domain separation: EIP-712 typed data (day, digest, version, chainId).
- Anti-replay: unique day, chainId in message, reject duplicates.
- Access: signer set updatable via governance (multisig), Pausable contract, RBAC.
- Upgrade: upgradeable proxy + governance multisig.

Client flow (dApp-side verification)
1) Download day's attestation JSON from repository/IPFS.
2) Calculate digest and compare with AICPOracleV2.digest(day).
3) If matches, use fields (CR, S, etc.) for mint policy UI.

- AICP is **utility token** for service consumption; no return promise; financial uses subject to local regulation.
- **Data and rules** publication for independent audit.

---

## 9. Roadmap
- **R1 (hardening):** attestation signing; on-chain hash; per-zone consistency tests; alerting.
- **R2 (market):** compute redemption integration (API), booking queue, per-user limits.
- **R3 (dynamic pricing):** \(P_E\) basket (day-ahead €/MWh EU average) and/or intraday dynamics.

---

## 10. Appendices
### 10.1 File formats
- `grid_load.csv`: `date, grid_wh_dc, grid_wh_dc_commit`
- `grid_by_zone_YYYY-MM-DD.csv`: `name,eic,wh`
- `grid_summary_YYYY-MM-DD.json`: `{day, zones_ok, wh_dc_raw, wh_dc_commit, alpha, zones:[...]}`
- `attestation_YYYY-MM-DD.json`: `{day, wh_dc_raw, wh_dc_commit, eur_per_mwh, alpha, stock_value_eur, S_base, U_hat, S, S_target_for_CR_min, mint_suggested, target_eur_per_aicp, CR, CR_min, mint_daily_cap_frac, [zones_ok]}`

### 10.2 Example parameters
- \(\alpha = 0{,}18\), \(P_E = 90\,€/\text{MWh}\), \(P_T = 1{,}50\,€/\text{AICP}\), \(\text{CR}_{\min}=1{,}10\), **cap** = 5% of \(S\) per day.

### 10.3 Glossary
- PTU: Program Time Unit, period duration (15/30/60 min).
- EIC: Energy Identification Code.
- Bidding Zone: market area in ENTSO-E.
- Coverage Ratio (CR): ratio between reserve value and nominal supply value.

### 10.4 Appendix A — EIC list and zones.csv
For readability and operational security, the complete list of EIC codes and respective timezones are not included in the whitepaper body.
- The reference file is data/zones.csv in the repository (format: name,eic,timezone).
- Example (excerpt):
  DE,10Y1001A1001A83F,Europe/Berlin
  FR,10YFR-RTE------C,Europe/Paris
  SE1,10YSE-1--------K,Europe/Stockholm
- The repository maintains file versioning and hash for audit.

- **PTU**: Program Time Unit, period duration (15/30/60 min).  
- **EIC**: Energy Identification Code.  
- **Bidding Zone**: market area in ENTSO-E.  
- **Coverage Ratio (CR)**: ratio between reserve value and nominal supply value.

---

**Note for reviewers:** this draft is aligned with current script code. Following review we will insert: (i) attestation hash/signature, (ii) Oracle v2 schema, (iii) complete numerical examples with a real day.