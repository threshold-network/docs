---
description: >-
  This page describes a technical diagram for v1 of the Verifiable Bitcoin
  Account, also called Milestone 1 ("M1"). A second version (v2) is planned to
  ship later.
---

# Technical Diagram

## Verifiable Bitcoin Accounts M1 Technical Diagram

{% hint style="info" %}
**The following features are planned and are NOT available in M1:**

* In-kind redemption: return the reserved bitcoin itself, for the whole position.
* Partial in-kind redemption: redeem part of a position; the rest stays reserved.
* Watchtower review for reserved redemptions, as for standard tBTC redemptions.
* Term renewal: extend the custody term of a position.
* Dissolution: after the term and a delay, release a position so its wallet is no longer tied to it; the owner keeps their tBTC as an ordinary claim.
* A lifetime limit on how much re-anchor fees can shrink one position.

_M1 positions carry over to v2 with the dates recorded at acceptance._&#x20;
{% endhint %}

In M1, a **Verifiable Bitcoin Account (VBA)** is a UTXO reservation in the tBTC v2 Bridge. When bitcoin is deposited into a tBTC threshold wallet and revealed to the Bridge, with the reservation vault named as its vault, it is never combined with other deposits: it retains its own _UTXO (the "anchor")_, held by the same wallet that custodies pooled tBTC. tBTC is minted against that anchor, one position at a time. _**The engineering name is a "reservation"**_<mark style="color:violet;">**;**</mark> this page uses both.

Every move of an anchor is a Bitcoin transaction with exactly one input and one output, proven to Ethereum with the Bridge's standard SPV checks. Custody security is the same as tBTC's: an honest majority of the threshold signers, with fraud challenges and stake slashing. The sections below show where each rule lives, how a position moves through its lifecycle, and how to verify any of it yourself.

***

### Pooled tBTC and a Verifiable Bitcoin Account

```
  POOLED DEPOSIT                          RESERVED DEPOSIT
  +----------------+                     +----------------+
  | deposits 1..n  |                     |    deposit     |
  +-------+--------+                     +-------+--------+
          |  sweep                               |  reveal
          v                                      v
  +----------------+                     +----------------+
  | shared main    |                     |    reserved    |
  | UTXO (pooled)  |                     |    deposit     |
  +----------------+                     +-------+--------+
                                                 |  acceptance
                                                 |  proves anchor
                                                 v
                                         +----------------+
                                         |  anchor UTXO   |
                                         |  (own output,  |
                                         |  P2WPKH wallet)|
                                         +----------------+
                                         
 ** Both sit under the same tBTC threshold wallet key
 ** A reserved deposit is excluded from every sweep
```

**What stays the same as ordinary tBTC**:

* The deposit and reveal flow is identical; a reserved deposit just names the reservation vault at reveal.
* Custody is the same threshold wallet and signing quorum as pooled tBTC (51 of 100 seats on mainnet).
* The tBTC you receive is ordinary tBTC: freely transferable.
* Fraud protection is identical: proven spends are honest, others stay challengeable, and a failed challenge terminates the wallet and slashes operators.

**What is different:**

* Bitcoin is never swept into the wallet's main UTXO; the Bridge and signers reject any sweep that includes a reserved deposit. The flag is permanent.
* Your position has its own traceable lineage: deposit UTXO, then anchor, then each re-anchor, each a one-input/one-output transaction.
* tBTC is minted against your anchor by the vault, which keeps an initiation fee in reserve to fund future re-anchor miner fees.

***

### Components and roles

<pre><code><strong>                                        PEOPLE
</strong>  +------------------------+       +----------------------------------------------+
  |   Owner (depositor)    |       |             Threshold Council                |   
  +------------------------+       +----------------------------------------------+
      |                                |                            |
      | reveal deposit,                | parameters, caps           | fee settings
      | request acceptance             | (governance delay)         | (instant)
      v                                v                            v
                                       
<strong>                                       ETHEREUM
</strong>  +--------------------------------------+          +-----------------------------+
  | tBTC v2 Bridge with reservation code |          | ReservationVault mints tBTC |
  | records positions, authorizes every  |  credit  | and pays the owner, minus   |
  | move, checks caps and Bitcoin proofs |  ----->  | the initiation fee; its fee |
  | keeps the claim equal to the anchor  |          | reserve pays re-anchor      |
  |                                      |          | miner fees                  |
  +--------------------------------------+          +-----------------------------+   
      ^                ^              ^
      | check          | prove        | notify: timeouts,
      | authorization  | Bitcoin txs  | stale deposits, stranding
      |                |              |
<strong>           OFF-CHAIN (signer operators)
</strong>  +-------------+ +--------------+ +------------+
  | tBTC signer | |     SPV      | |  watchers  |
  |    nodes    | |  maintainers | |            |
  +-------------+ +--------------+ +------------+
      |
      | sign anchor and re-anchor transactions
      v
<strong>    BITCOIN
</strong>    deposit UTXO --> anchor UTXO --> next anchor UTXO (1 in, 1 out each)
</code></pre>



<table data-search="false"><thead><tr><th>Role</th><th>What it does</th><th>What it cannot do</th></tr></thead><tbody><tr><td>Owner (depositor)</td><td>Sends and reveals the deposit; requests acceptance; receives the tBTC</td><td>Move, redeem or close the position in M1; spend the anchor</td></tr><tr><td>tBTC threshold wallet and its signer nodes</td><td>Custodies the anchor; proposes and signs anchor and re-anchor transactions</td><td>Include a reserved deposit in any sweep; sign a transaction outside a pending Bridge authorization (a fraud challenge punishes it)</td></tr><tr><td>SPV maintainers</td><td>Prove confirmed anchor and re-anchor transactions to the Bridge</td><td>Forge a proof or change what a proof settles; only authorized maintainers can submit proofs</td></tr><tr><td>Watchers (signer software)</td><td>File the permissionless housekeeping calls: action timeouts, stale deposits, stranding</td><td>Grant, block or speed anything up; all such calls are open to anyone</td></tr><tr><td>Threshold Council (Bridge governance and vault owner)</td><td>Sets parameters and caps with a governance delay; changes the vault's initiation fee (hard-capped at 5%) and fee reserve target instantly; requests re-anchors off healthy Live wallets</td><td>Change values in ways the Bridge rejects; move a position without a signed, proven transaction</td></tr><tr><td>Anyone</td><td>Repay the vault's public fee debt; request re-anchors from retiring wallets; file housekeeping notifications</td><td>Request acceptance for someone else's deposit; re-anchor a position off a Live wallet (governance only); send a position anywhere except another Live tBTC wallet</td></tr></tbody></table>

***

### Where each rule is enforced

<table data-search="true"><thead><tr><th>Rule</th><th>Where</th><th>How</th></tr></thead><tbody><tr><td>Only the wallet's key can spend an anchor</td><td>Bitcoin</td><td>The anchor is a P2WPKH (native SegWit) output of the tBTC wallet; Bitcoin checks the wallet's signature and nothing else. A reserved deposit's refund locktime also protects the pre-acceptance path: an unaccepted deposit always becomes refundable on Bitcoin.</td></tr><tr><td>Every move is authorized in advance</td><td>Bridge (Ethereum)</td><td>Each move starts as a pending authorization (Acceptance or Reanchor) with a deadline and snapshots of the parameters that matter. No transaction settles without its authorization.</td></tr><tr><td>Caps hold</td><td>Bridge</td><td>Single-reservation, per-wallet count, per-wallet amount, global count and global amount caps are all checked when an action is requested, and capacity is reserved until settlement.</td></tr><tr><td>Exact transaction shape</td><td>Bridge</td><td>At proof time the Bridge verifies the SPV proof and checks the Bitcoin transaction: exactly one input, exactly one output paying the wallet's key, fee within the snapshotted maximum, amount within the rules.</td></tr><tr><td>Claim equals anchor</td><td>Bridge</td><td>The Bridge writes the minted claim and the anchor amount together, at acceptance and at every re-anchor; the two are never changed separately.</td></tr><tr><td>Signers only sign authorized moves</td><td>Signer nodes (off-chain)</td><td>Every signer re-checks the on-chain authorization before signing a reservation transaction; unauthorized proposals are hard-rejected.</td></tr><tr><td>Unauthorized signatures are punished</td><td>Ethereum (fraud challenges)</td><td>Proven anchor and re-anchor spends are recorded as honest; any other signature over an anchor remains challengeable, and a wallet that cannot defeat a challenge is terminated and its operators' stake slashed.</td></tr></tbody></table>

{% hint style="warning" %}
The rules do not live in a per-account Bitcoin script. _On the Bitcoin side, one fact holds:_ only the tBTC wallet's key can spend the anchor. Everything else that moves is allowed: to which wallet, how large, when, and under which caps is enforced by the Bridge's reservations extension on _Ethereum._ The Bitcoin transaction is proven to Ethereum; Ethereum decides whether it counts.
{% endhint %}

***

### Bitcoin transactions

```
  ANCHOR TRANSACTION (acceptance)
  +-------------------------------------+
  | input 0:  the reserved deposit UTXO |
  | output 0: P2WPKH, the wallet's key  |
  +-------------------------------------+
  value rules (proof-time):
    deposit - output <= max fee
    output >= minimum amount

  RE-ANCHOR TRANSACTION
  +-------------------------------------+
  | input 0:  the current anchor        |
  | output 0: P2WPKH, the target wallet |
  +-------------------------------------+
  value rules (proof-time):
    anchor - output <= max fee
    output > max fee (dust floor)
  (the request itself required: anchor > minimum + max fee)

  LINEAGE:
  deposit UTXO -> anchor on wallet A -> anchor on wallet B -> ...
  every transaction spends exactly the output created by the previous
  one, so any Bitcoin node or explorer can re-walk the whole chain
  from the original deposit.
```

Both shapes are enforced at proof time by the Bridge: exactly one input, exactly one output, the output paying the authorized wallet's key, fee within the snapshotted maximum, and the value floor for that transaction type; the minimum-plus-max-fee floor for re-anchors is checked when the re-anchor is requested. The signer software builds P2WPKH outputs. A Bitcoin node can verify that the signature belongs to the wallet's key, but it cannot verify anything further: the one-input/one-output shape and the value rules are Ethereum checks applied to the proven transaction.

***

### Amounts and fees

```
  ILLUSTRATIVE NUMBERS (not the deployed parameters)

  deposit           10.0000000 BTC
    |
    |  anchoring miner fee 0.0001 BTC, paid by the owner
    v
  anchor             9.9999000 BTC
    |
    |  tBTC minted 1:1 against the anchor
    v
  tBTC minted        9.9999000 tBTC
    |
    |  initiation fee 0.40% = 0.0399996 tBTC, kept in the vault reserve
    v
  owner receives     9.9599004 tBTC

  later re-anchor, miner fee 0.0001 BTC:
  anchor 9.9999 -> 9.9998 BTC; the vault burns 0.0001 tBTC from its
  reserve; the owner's 9.9599004 tBTC is untouched.
```

* Who pays each fee: the anchoring miner fee is paid by the owner (the anchor is the deposit minus the fee, and tBTC is minted on the anchor). The initiation fee is taken from the minted amount and kept by the vault. Re-anchor miner fees are paid from the vault's fee reserve: the vault burns tBTC in step with each bitcoin fee, so the total tBTC supply falls as bitcoin is spent.
* Public fee debt: if the reserve is short, the unpaid part is recorded as public debt. While the debt is non-zero, tBTC supply exceeds backing by exactly that amount. Anyone can repay it, by burning their own tBTC. The vault owner's fee sweep repays the debt first, and can only send what is above the reserve target.
* No normal deposit treasury fee applies to reserved deposits.
* There is no lifetime cap on re-anchor fees in M1 (one is planned for v2). Each move's fee is capped by the snapshotted maximum, and a re-anchor can only be requested while the anchor exceeds the minimum plus one max fee, so re-anchor fees can never take a position below the minimum.

***

### States

```
  RESERVED DEPOSIT (before a position exists)

  +---------+  acceptance proof settles  +--------------------------+
  | Pending | -------------------------> | Accepted: now a position |
  +---------+                            +--------------------------+
       |
       | refund deadline passed, no acceptance pending: anyone
       | marks it stale (governance may force it earlier)
       v
  +---------+
  | Stale   |  refund on Bitcoin; can never be accepted or swept
  +---------+

  RESERVATION POSITION

  +---------+
  | Unknown |  no position yet; a pending acceptance
  +---------+  lives only in its action record
       |
       | acceptance proof settles
       v
  +---------+  re-anchor requested    +---------------+
  | Active  | ----------------------> | ActionPending |
  |         | <---------------------- |               |
  +---------+  re-anchor proven (now  +---------------+
       |       on the target wallet)
       |       or timed out (still on
       |       the source wallet)
       |
       | custodying wallet Terminated or Closed,
       | or Closing past the dissolution date
       v
  +----------+
  | Stranded |
  +----------+

  A late proof of an already-signed re-anchor can restore a Stranded
  position onto the new wallet. Closed is v2 only (in-kind redemption
  or dissolution); it is unreachable in M1.

  ACTION STATES (one per request): Pending, then Settled, or TimedOut
  (a late proof can still settle it), or Superseded (an older request's
  late proof used the anchor first). Vetoed is v2 only. M1 action
  types: Acceptance and Reanchor.
```

***

### When things go wrong

<table data-search="true"><thead><tr><th>Situation</th><th>Who can act</th><th>Result</th></tr></thead><tbody><tr><td>Acceptance not signed in time</td><td>Anyone files the timeout; the depositor can request again</td><td>Capacity released; new request with fresh nonce while refund window open; no slashing.</td></tr><tr><td>Re-anchor not completed in time</td><td>Anyone files the action timeout</td><td>Target slot released; position back to Active on same source; cooldown on next non-governance request; no slashing.</td></tr><tr><td>Deposit never accepted</td><td>Anyone marks it stale after refund deadline; governance can force it earlier</td><td>Pending record cleared; bitcoin refunded via the deposit script refund path on Bitcoin.</td></tr><tr><td>Custodying wallet terminated</td><td>Anyone files the stranding</td><td>Capacity released, position Stranded. Owner's tBTC unchanged; no compensation.</td></tr><tr><td>Proof arrives late or after wallet stopped</td><td>SPV maintainer submits it</td><td>Timed-out acceptance provable within deadline plus term; re-anchor provable anytime; wallet already Closing/Closed/Terminated means stranded.</td></tr><tr><td>The custodying wallet's signers stop responding</td><td>No one can move the anchor without them</td><td>Positions stay where they are and the owner's tBTC is unaffected. If the wallet then misses a protocol deadline (for example while moving its funds) it is slashed and terminated, and its positions can be stranded. There is no owner-held key to recover the bitcoin independently.</td></tr></tbody></table>

The M1 reservation timeouts - acceptance and re-anchor - do not slash anyone; no stake is seized by either. Wallet-level timeouts are different and still apply: a retiring wallet that fails to move its funds in time is slashed and terminated, which is standard tBTC wallet machinery, separate from the reservation rules.

***

### Caps and parameters

All values below are set by the Threshold Council through Bridge governance: begin, governance delay, finalize. Vault fee settings change instantly, within the contract's hard cap.

| Parameter                         | Description                                                                                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `reservationMinAmount`            | Minimum anchor size, in satoshi. A deposit must be at least this plus the max fee, and a re-anchor can only be requested while the anchor exceeds this plus the max fee. |
| `reservationTxMaxFee`             | Maximum miner fee for one anchor or re-anchor transaction.                                                                                                               |
| `reservationTermSeconds`          | The custody term of a position; the contract accepts between 90 and 730 days.                                                                                            |
| `reservationDissolutionDelay`     | Delay after the term before dissolution can happen; a v2 feature, recorded in M1; in M1 it also sets when a position left on a Closing wallet can be stranded.           |
| `reservationActionTimeout`        | How long a requested action has before it times out.                                                                                                                     |
| `reservationRenewalWindowSeconds` | Renewal window; a v2 feature, must still be set.                                                                                                                         |
| `reservationMaxTotalAmount`       | Global cap on the satoshi locked under active reservations; 0 turns the cap off.                                                                                         |
| `maxReservationsPerWallet`        | How many open reservations one wallet may custody; must be greater than 0 (1 at launch).                                                                                 |
| `maxReservationsAmountPerWallet`  | Cap on the satoshi one wallet may custody across its reservations; 0 turns it off.                                                                                       |
| `reservationMaxSingleAmount`      | Cap on a single position; 0 turns it off.                                                                                                                                |
| `maxActiveReservations`           | Global cap on open positions; must be greater than 0.                                                                                                                    |

Enforced relations: max fee > 0; minimum > max fee; term 90-730 days; renewal window > 0 and < term; action timeout > 2 hours; per-wallet count > 0; total cap <= max open positions x single cap (when both set); vault address can only change when no reservation is open and no reserved deposit is pending.

Launch posture: M1 launches with a small total cap, limited to design partners, with `maxReservationsPerWallet` set to 1 and a planned custody term of 12 months. Quotable code defaults: initiation fee 0.40% at deploy, hard-capped at 5%; minimum deposit age 2 hours; refund safety margin 24 hours; term range 90-730 days; signers wait 6 Bitcoin confirmations before proposing. The other values above are governance-set.

***

### Trust model

<table data-search="true"><thead><tr><th>Component</th><th>What you rely on</th><th>Enforced by</th></tr></thead><tbody><tr><td>Bitcoin custody of the anchor</td><td>An honest majority of the custodying wallet's signers (51 of 100 seats on mainnet), the same as pooled tBTC</td><td>Threshold ECDSA; fraud challenges and stake slashing</td></tr><tr><td>Correct reservation records and minting</td><td>Bitcoin and Ethereum consensus only</td><td>The Bridge checks the SPV proof and the exact transaction shape before any state change</td></tr><tr><td>Segregation from pooled funds</td><td>Nothing extra</td><td>The Bridge rejects sweeps containing reserved deposits; signers refuse them too</td></tr><tr><td>Deposit before acceptance</td><td>Nothing extra</td><td>The standard tBTC deposit refund path on Bitcoin after the refund locktime</td></tr><tr><td>Proof delivery</td><td>Authorized SPV maintainers, and the Bridge's Bitcoin relay being kept up to date</td><td>Liveness only: maintainers cannot forge a proof</td></tr><tr><td>Moving positions off retiring wallets</td><td>tBTC signer software doing its duties; free slots on Live wallets</td><td>Permissionless requests, caps, timeouts</td></tr><tr><td>Housekeeping (timeouts, stale deposits, stranding)</td><td>Anyone</td><td>Permissionless calls, automated by signer software</td></tr><tr><td>Caps and parameters</td><td>The Threshold Council</td><td>Bridge governance with delay; snapshots protect in-flight actions</td></tr><tr><td>Vault fees</td><td>The Threshold Council as vault owner</td><td>Instant changes, contract cap of 5%</td></tr></tbody></table>

**Net trust model:** M1 keeps tBTC v2's custody model. What it adds is segregation and a traceable, proven lineage for each deposit. There is no depositor-held key and no Bitcoin Script recovery path after acceptance.

***

### M1 limits

1. No in-kind redemption yet. In M1, the owner holds, transfers, or uses tBTC, or redeems through standard (pooled) tBTC redemption, which pays from pooled wallet funds, not the anchor.
2. No owner-initiated way to close a position in M1: no redemption, renewal or dissolution. It stays open until v2 or until its wallet dies (Stranded).
3. The term is recorded, not enforced. Expiry and dissolution dates exist so v2 can honor the term; in M1, their only effect is the stranding gate on Closing wallets.
4. Stranding: if the custodying wallet is terminated, the in-kind option is lost; the tBTC balance remains unchanged; no compensation is available. If that wallet's bitcoin is really lost, the shortfall affects tBTC backing as a whole.
5. Moving positions depends on the network: re-anchor needs a free slot on another Live wallet and working signer software; off a healthy Live wallet, only governance can move a position.
6. Re-anchor fees shrink the anchor, not the owner's tBTC; covered by the vault's fee reserve, with shortfalls becoming public repayable debt. No lifetime fee cap in M1; positions at or below minimum plus one max fee can never be re-anchored. Size deposits well above the minimum.
7. Capacity is checked at acceptance request, not when bitcoin is sent. Check free slots and caps first; unaccepted deposits are refunded in Bitcoin after their locktime.
8. Launch is limited to design partners and small caps, with one reservation per wallet.
9. Fees and parameters are set by the Threshold Council. Vault changes are instant (capped at 5%); Bridge changes are subject to the governance delay and never affect in-flight actions or recorded dates.

***

### Verify a reservation yourself

Everything above is public state; no special tooling is required.

1. **Compute the reservation key:** the standard tBTC deposit key - keccak256 of the funding transaction hash and output index, as the Bridge uses for every deposit.
2. **Read the Bridge address:** `reservations(key)` gives owner, wallet, anchor, amounts, state, and dates; `reservationActions(key, nonce)` for pending actions; `reservationParameters()` for operational parameters; `reservationCaps()` for the per-wallet amount, single-position, and max-open-positions caps; `activeReservationsCount()` for current open count; `walletReservationsCount(hash)` and `walletReservationsAmount(hash)` per wallet; `pendingReservedDeposits()` and `reservedDepositWallet(key)` pre-acceptance; `reservationByAnchorUtxo(txHash, 0)` finds a position from its Bitcoin outpoint.
3. **Check the anchor output** **on any Bitcoin node or explorer:** it must pay the custodying wallet's key at the recorded amount.
4. **Follow the events:** `ReservationAcceptanceRequested`, `ReservationAccepted`, `ReservationReanchorRequested`, `ReservationReanchored`, `ReservationAcceptanceTimedOut`, `ReservationReanchorTimedOut`, `ReservationLateSettled`, `ReservationStranded`, `ReservedDepositMarkedStale`.
5. **Read the vault:** `initiationFeeBps`, `feeReserveTarget`, and `inKindFeeDebtSat` (the outstanding public fee debt).
