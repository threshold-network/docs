# Technical Diagram

Verifiable Bitcoin Accounts (VBA) combine three layers: the institution's existing custody, Threshold Network's distributed signer infrastructure, and pre-approved onchain deployment venues. Bitcoin Script enforces all spending conditions at the consensus layer.

<figure><img src="../.gitbook/assets/Verifiable Bitcoin Accounts VBA Lifecycle Diagram.png" alt="Verifiable Bitcoin Accounts (VBA) Lifecycle by Threshold Network"><figcaption></figcaption></figure>

## Layer 1: Custody Compatibility

Any key management system that can sign Bitcoin transactions and produce BIP-174 (PSBT) compatible partial signatures can participate as depositor (D), custodian (C), or signer (S).

```
       +--------------------------------------------------------------------------+
       |                           CUSTODY COMPATIBILITY                          |
       +--------------------------------------------------------------------------+

       +----------------------+  +----------------------+  +----------------------+
       | QUALIFIED CUSTODIAN  |  |    THRESHOLD MPC     |  |     SELF-CUSTODY     |
       | (holds key C or D)   |  |    (holds key S)     |  |     (holds key D)    |
       |                      |  |                      |  |                      |
       | - Any regulated      |  | - tECDSA threshold   |  | - HSM-based signing  |
       |   custodian that     |  |   signing (e.g.,     |  | - Hardware wallets   |
       |   supports PSBT      |  |   keep-core nodes)   |  | - MPC wallets        |
       | - MPC or HSM-based   |  | - Produces single    |  |                      |
       |   key management     |  |   aggregated pubkey  |  |                      |
       |                      |  |   for Bitcoin Script |  |                      |
       +----------------------+  +----------------------+  +----------------------+
                  |                         |                         |
                  +-------------------------+-------------------------+
                                            |
                                            v
                      +-----------------------------------------+
                      |         PSBT SIGNING INTERFACE          |
                      |          (BIP-174 Compatible)           |
                      |     Each party signs independently      |
                      |    and contributes partial signature    |
                      +-----------------------------------------+
                                            |
                                            v
                        +--------------------------------------+
                        |     FINALIZED BITCOIN TRANSACTION    |
                        |  Broadcast + SPV-proven to Ethereum  |
                        +--------------------------------------+
```

#### Template-to-role mapping

<table><thead><tr><th width="118.2135009765625">Template</th><th width="147.16143798828125">Roles present</th><th>Required signers per path</th></tr></thead><tbody><tr><td>qc_v1</td><td>D + C + S</td><td>COOPERATIVE: D+C+S; MIGRATION: S+C; EARLY_EXIT: D+C; LAST_RESORT: D</td></tr><tr><td>self_v1</td><td>D + S (no C)</td><td>COOPERATIVE: D+S; MIGRATION: S; LAST_RESORT: D</td></tr></tbody></table>

**Key principle:** No asset migration is required. The custody relationship that the institution already maintains continues to govern the asset throughout the VBA's lifecycle.

***

## Layer 2: Multi-Party Control Model

```
 +---------------------------------------------------------------------------------+
 |                             MULTI-PARTY CONTROL FLOW                            |
 +---------------------------------------------------------------------------------+

 +-----------------------+    +------------------------+    +----------------------+
 |      INSTITUTION      |    |     SIGNER NETWORK     |    |      CUSTODIAN       | 
 |     Signing Key D     |    |      Signing Key S     |    |     Signing Key C    |
 |      (depositor)      |    | (single aggregated key |    |     (qc_v1 only)     |
 |                       |    | may be tECDSA-managed  |    |                      |
 |                       |    |       off-chain)       |    |                      |
 +-----------------------+    +------------------------+    +----------------------+
            |                             |                             |
            |                             |                             |
            +-----------------------------+-----------------------------+
                                          |
                                          v         
              +-------------------------------------------------------+
              |                    PATH SELECTION                     |
              |                                                       |
              |     COOPERATIVE:  D + C + S (3-of-3 CHECKMULTISIG)    |
              |     MIGRATION:    S + C     (2-of-2 CHECKMULTISIG)    |
              |     EARLY_EXIT:   D + C     (2-of-2 CHECKMULTISIG)    |
              |     LAST_RESORT:  D only    (1-of-1 CHECKSIG)         |
              +-------------------------------------------------------+
                                          |
                                          v
              +-------------------------------------------------------+
              |               SBT CONSTRUCTION (BIP-174)              |  
              |                                                       |
              |  Each signer adds partial sig to PSBT independently   |
              +-------------------------------------------------------+
                                          |
                                          v
              +-------------------------------------------------------+
              |                   PSBT FULLY SIGNED                   |  
              |                                                       |
              |                Broadcast to Bitcoin network           |
              +-------------------------------------------------------+
                                          |
                                          v                    
              +-------------------------------------------------------+
              |                   SPV PROOF SUBMITTED                 |  
              |                                                       |
              |             CovenantBitcoinTxProofVerifier            |
              |               validates proof on Ethereum             |
              |         CovenantAuthority updates reserve state       |
              +-------------------------------------------------------+
                
```

**Enforcement:** The quorum threshold is encoded in the Bitcoin Script governing the UTXO. Any transaction below the threshold is rejected by the Bitcoin network at the consensus layer.

***

## Layer 3: Bitcoin Script Enforcement

Spending conditions are written directly in Bitcoin Script and validated by every Bitcoin full node. &#x20;

```
+-------------------------------------------------------------------------------+
|                    BITCOIN SCRIPT ENFORCEMENT MODEL (qc_v1)                   |
+-------------------------------------------------------------------------------+

VBA UTXO (P2WSH with embedded witness script)
|
+-> SPENDING PATH 1: COOPERATIVE -- Unanimous authorized movement
|   |
|   +-- 3-of-3 OP_CHECKMULTISIG (D + C + S all required)
|   +-- No locktime constraint
|   +-- No destination restriction in Bitcoin Script *
|
+-> SPENDING PATH 2: MIGRATION -- Post-maturity signer-led move
|   |
|   +-- 2-of-2 OP_CHECKMULTISIG (S + C)
|   +-- OP_CHECKLOCKTIMEVERIFY: locktime = maturityHeight
|   +-- No destination restriction in Bitcoin Script *
|
+-> SPENDING PATH 3: EARLY_EXIT -- Pre-maturity depositor + custodian
|   |
|   +-- 2-of-2 OP_CHECKMULTISIG (D + C)
|   +-- OP_CHECKLOCKTIMEVERIFY: locktime = maturityHeight - beta
|   +-- No destination restriction in Bitcoin Script *
|
+-> SPENDING PATH 4: LAST_RESORT -- Solo depositor recovery
    |
    +-- OP_CHECKSIG: single signature (D only)
    +-- OP_CHECKLOCKTIMEVERIFY: locktime = maturityHeight + delta2
    +-- Activates if all other parties are unavailable
```

Destination address constraints are enforced by Covenant Authority on Ethereum, NOT by Bitcoin Script. Bitcoin Script only checks signatures and timelocks.

_Validated by: Every Bitcoin full node (script validity only)_

#### self\_v1 Template (Self-custody, no custodian)

```
+-----------------------------------------------------------------+
|           BITCOIN SCRIPT ENFORCEMENT MODEL (self_v1)            |
+-----------------------------------------------------------------+

Keys: D = depositor, S = signer (no custodian in this template)

+-> SPENDING PATH 1: COOPERATIVE
|   +-- 2-of-2 OP_CHECKMULTISIG (D + S)
|   +-- No locktime
|
+-> SPENDING PATH 2: MIGRATION
|   +-- OP_CHECKSIG: single signature (S only)
|   +-- OP_CHECKLOCKTIMEVERIFY: locktime = maturityHeight
|
+-> SPENDING PATH 3: LAST_RESORT
    +-- OP_CHECKSIG: single signature (D only)
    +-- OP_CHECKLOCKTIMEVERIFY: locktime = maturityHeight + delta2
```

**No external dependencies:** The rules governing each VBA do not rely on application-layer software, smart contracts on other chains, or proprietary policy engines.

***

### Liquidation & Migration Path

When a position is liquidated in an approved venue, Threshold signers automatically execute the predefined migration path.

```
  ┌──────────────────────────────────────────────────────────┐
  │              LIQUIDATION & MIGRATION FLOW                │
  └──────────────────────────────────────────────────────────┘

  ┌──────────────────────┐
  │    ONCHAIN VENUE     │   Liquidation event triggered
  │   (Aave, Morpho,     │   (e.g., LTV breach, oracle price)
  │  Curve, Yield Basis) │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │   THRESHOLD SIGNERS  │   Independent verification
  │                      │   of onchain state
  │ • Direct read of     │
  │   contract state     │
  │ • No external oracle │
  │ • No off-chain       │
  │   coordination       │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │   QUORUM EVALUATION  │   Threshold signers reach
  │                      │   quorum agreement
  │ Each signer verifies │
  │  conditions are met  │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │    PRE-CONSTRUCTED   │   Migration PSBT signed
  │     MIGRATION PSBT   │   by quorum
  │                      │
  │     Destination:     │
  │    Pre-defined at    │
  │    account setup     │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │     BROADCAST TO     │
  │   BITCOIN NETWORK    │
  └──────────┬───────────┘
             │
        ┌────┴────────────────────────────┐
        ▼                                 ▼           
  ┌──────────────────────┐  ┌───────────────────────────┐
  │   THRESHOLD BRIDGE   │  │       PRE-APPROVED        │
  │      (Default)       │  │    COUNTERPARTY ADDRESS   │
  │                      │  │  (Where venue settlement  │
  │                      │  │         requires it)      │
  └──────────────────────┘  └───────────────────────────┘
```

**Critical property:** The migration destination is whitelisted at account setup. Signers do not exercise discretion about where capital goes. They verify a predefined condition and execute a predefined action.

***

### Recovery Architecture

Every VBA includes layered recovery paths that operate without requiring counterparty cooperation.

```
+-----------------------------------------------------------------------+
|                         RECOVERY ARCHITECTURE                         |
+-----------------------------------------------------------------------+


FAILURE SCENARIO 1: Signer network disrupted
|
+-- Recovery Path A: Joint depositor + custodian recovery (EARLY_EXIT)
|   |
|   +-- Threshold signers NOT required
|   +-- Depositor key (D) + custodian key (C) sign
|   +-- 2-of-2 OP_CHECKMULTISIG on Bitcoin Script
|   +-- REQUIRES a timelock: locktime = maturityHeight - beta
|       (activates BEFORE full maturity, but still time-gated)   
+-- Reserve state must be in EXITING on Ethereum (CovenantAuthority)


FAILURE SCENARIO 2: Both signer network AND custodian are unavailable
|
+-- Recovery Path B: Depositor solo recovery (LAST_RESORT)
|    |
|    +-- Only the institution's key (D) required
|    +-- OP_CHECKSIG (not multisig -- single signature)
|    +-- Enforced by OP_CHECKLOCKTIMEVERIFY
|    +-- Locktime = maturityHeight + delta2
|        (longest timelock, set at account creation) 
+-- Activates automatically when the timelock expires
```

#### Enforcement Mechanism:

Recovery paths encoded in Bitcoin Script (P2WSH).  However, the reserve state machine lives on Ethereum: CovenantAuthority.reserveStates is the canonical record. IBitcoinRelay must be live for freshness check, Attester quorum (CovenantOracle) required for state advances, Bitcoin txs must be SPV-proven back to Ethereum

***

### Complete Lifecycle Summary

A complete view of how a VBA progresses from setup through operation to potential recovery.

```
  ┌────────────────────────────────────────────────────┐
  │                   VBA LIFECYCLE                    │
  └────────────────────────────────────────────────────┘

  PHASE 1: SETUP
  ┌────────────────────────────────────────────────────┐
  │ • Quorum structure defined                         │
  │ • Whitelisted venues selected                      │
  │ • Recovery paths configured with timelocks         │
  │ • Bitcoin Script encoded                           │
  │ • PSBTs pre-constructed                            │
  │ • Optional ACA executed                            │
  └─────────────────────┬──────────────────────────────┘
                        ▼
  PHASE 2: FUNDING
  ┌────────────────────────────────────────────────────┐
  │ • Institution funds VBA from existing custody      │
  │ • UTXO created with embedded Script conditions     │
  │ • Account verified and operational                 │
  └─────────────────────┬──────────────────────────────┘
                        ▼
  PHASE 3: DEPLOYMENT
  ┌────────────────────────────────────────────────────┐
  │ • Capital deployed into whitelisted venues         │
  │ • Multi-party authorization on every movement      │
  │ • All transactions verifiable on Bitcoin L1        │
  └─────────────────────┬──────────────────────────────┘
                        ▼
  PHASE 4: ACTIVE OPERATION
  ┌────────────────────────────────────────────────────┐
  │ • Threshold signers monitor venue activity         │
  │ • Yield accrues to the account                     │
  │ • Standard reporting generated                     │
  │ • Quarterly reviews with coverage team             │
  └─────────────────────┬──────────────────────────────┘
                        ▼
  PHASE 5: EXIT (one of three paths)
  ┌────────────────────────────────────────────────────┐
  │ Path A: Normal Exit                                │
  │   Multi-party quorum unwinds positions             │
  │   Capital returns to the institution's custod      │
  │                                                    │
  │ Path B: Liquidation-Triggered Migration            │
  │   Pre-defined migration path executes              │
  │   Capital settles on the whitelisted destination   │
  │                                                    │
  │ Path C: Recovery (Counterparty Failure)            │
  │   Recovery path activates per Script conditions    │
  │   The institution recovers capital independently   │
  └────────────────────────────────────────────────────┘
```

***

### Trust Model Summary

<table><thead><tr><th width="207.91668701171875">Component</th><th width="235.427001953125">Trust Assumption</th><th width="291.43231201171875">Enforcement Mechanism</th></tr></thead><tbody><tr><td>Spending conditions</td><td>None required</td><td>Bitcoin Script + consensus</td></tr><tr><td>Quorum threshold</td><td>None required (for Bitcoin Script rules)</td><td>Bitcoin Script (OP_CHECKMULTISIG); threshold varies by path</td></tr><tr><td>Timelock enforcement</td><td>None required</td><td>Bitcoin Script (OP_CHECKLOCKTIMEVERIFY)</td></tr><tr><td>Reserve state machine</td><td>IBitcoinRelay liveness</td><td>CovenantAuthority contract on Ethereum (canonical state)</td></tr><tr><td>Attestation quorum</td><td>Threshold attester honesty (threshold-bound)</td><td>CovenantOracle on Ethereum; ECDSA signature quorum required</td></tr><tr><td>Migration execution</td><td>Threshold signer honesty (threshold-bound)</td><td>Bitcoin Script path + CovenantAuthority state transition</td></tr><tr><td>Recovery activation</td><td>None required (Bitcoin Script paths)</td><td>Bitcoin consensus + OP_CHECKLOCKTIMEVERIFY</td></tr><tr><td>Legal enforceability</td><td>Counterparty contractual compliance</td><td>Tripartite Account Control Agreement (optional)</td></tr></tbody></table>

**Net trust model:** The institution does not need to trust Threshold Network for the integrity of the rules governing its capital. It uses Threshold's signer network for operational liveness. If that liveness fails, the recovery path activates without requiring Threshold's participation.

***

### Verification

Every condition governing a VBA can be verified independently:

* **Spending conditions:** Inspect the Bitcoin Script attached to the UTXO using any Bitcoin full node or open-source script debugger
* **Quorum requirements:** Validate the multisig structure encoded in Script
* **Recovery paths:** Confirm the timelock parameters and signing requirements
* **Onchain state:** Read the UTXO and transaction history directly from any Bitcoin full node
* **Deployment activity:** Audit transactions on the relevant chain and reconcile against the VBA's signed PSBT history

No proprietary tooling is required for any of this verification.
