---
description: >-
  This page describes the lifecycle and process of using Verifiable Bitcoin
  Accounts (VBA) M1 from product activation to concluding the position.
---

# VBA Lifecycle

## Lifecycle of Verifiable Bitcoin Accounts (VBA)

```
   +---------------+
   |   Activation  |  governance switches M1 on: vault trusted, caps set
   +-------+-------+
           |
           v
   +----------------+
   |   Deposit and  |  bitcoin sent to the tBTC wallet, revealed
   |     Reveal     |  to the reservation vault
   +-------+--------+
           |
           v
   +----------------+
   |   Acceptance   |  depositor requests on Ethereum; Bridge
   |    Request     |  validates and reserves capacity
   +-------+--------+
           |
           v
   +----------------+
   |  Anchoring on  |  one Bitcoin transaction moves the deposit
   |    Bitcoin     |  into its own anchor UTXO
   +-------+--------+
           |
           v
   +----------------+
   |   Proof and    |  Bridge writes the position and credits the
   |    Minting     |  vault; the vault mints tBTC, pays the owner
   +-------+--------+
           |
           v
   +----------------+       +------------------+
   |    Custody     | --->  |     re-anchor    |
   | (position open)| <---  |   (repeatable)   |
   |    Active      |       +------------------+
   +-------+--------+        when a wallet retires,
           |                 the anchor moves to a
           |                 different Live wallet
           v
   M1 end state: a position ends only if its custodying wallet dies
   (Stranded). Exits such as in-kind redemption, renewal and
   dissolution arrive in v2.
```

{% stepper %}
{% step %}
### Activation

M1 is switched on by governance. The reservation machinery ships inert: the router is registered with the Bridge, the vault is deployed but untrusted, and deposits cannot be revealed to it. Governance sets the parameters and caps through the delayed governance process, then marks the vault as trusted. That final switch is when reservations become possible. Signer software enables reservation duties only on networks where M1 is activated.
{% endstep %}

{% step %}
### Deposit and Reveal

Send bitcoin to a tBTC threshold wallet `deposit script` then reveal the deposit on Ethereum. A deposit becomes reserved by naming the reservation at reveal time; the wallet named and committed to in the deposit script is the designated wallet, and only it can accept the deposit. _Reserved deposits must use a refund locktime no later than the reveal time plus the term plus 24 hours,_ so an unaccepted deposit always becomes refundable on Bitcoin. No reservation minimum or cap is checked at reveal: only the ordinary dust threshold.

{% hint style="warning" %}
Capacity is checked when acceptance is requested, not when the bitcoin is sent. Before sending, check that the designated wallet has a free slot under its caps and that the global caps have room. Otherwise, the acceptance request will fail, and you will have to wait for the bitcoin to become refundable through the deposit script's refund path.&#x20;
{% endhint %}

Reserved deposits are not charged the Bridge's normal deposit treasury fee; the vault's initiation fee applies instead [(see Amounts and fees).](https://docs.threshold.network/verifiable-bitcoin-accounts/technical-diagram#amounts-and-fees)
{% endstep %}

{% step %}
### Acceptance: request, then proof

```
  1. DEPOSITOR, on Ethereum: requests acceptance for the deposit.
       The Bridge checks the deposit (revealed, reserved, routed to the
       vault, not swept, no duplicate position or pending action), checks
       that the named wallet is the designated wallet and is Live, checks
       the amount against the minimum and the caps, and records a pending
       authorization: a deadline, snapshots of the max fee, the minimum
       amount, the term and the dissolution delay. Capacity is reserved.
  2. DESIGNATED WALLET, on Bitcoin: the wallet's signers check the
       authorization on-chain, wait for the deposit to be confirmed, and
       sign a one-input/one-output anchor transaction that pays the
       wallet's key.
  3. SPV MAINTAINER, on Ethereum: proves the confirmed anchor transaction
       to the Bridge (Merkle inclusion, coinbase proof, proof-of-work).
  4. BRIDGE, on Ethereum: settles the action - writes the position (owner,
       custodying wallet, anchor outpoint, claim equal to anchor amount,
       state Active, expiry and dissolution dates recorded) and credits
       the vault. The vault mints tBTC against the anchor and transfers it
       to the owner, minus the initiation fee.
```

**Only the depositor may request acceptance.** The signing window is bounded: the deposit must be at least 2 hours old, signers must wait for _6 Bitcoin confirmations_ before proposing, and the window must fit within the refund deadline minus a 24-hour safety margin. Parameters that matter for proof and settlement are snapshotted at the time of the request; subsequent governance changes do not affect the in-flight action or the dates already recorded.
{% endstep %}

{% step %}
### Custody

The position is `Active`. The owner holds ordinary tBTC, which is the anchor amount minus the initiation fee. The anchor sits under the custodying wallet; the wallet's honest majority protects it, fraud challenges police it, and the Bridge keeps the tracked claim equal to the anchor at all times. Re-anchoring is available at any position age. The recorded term dates exist so v2 can honor the term; in M1, nothing expires, renews, or closes on them, except that a position still sitting on a Closing wallet can be stranded once its dissolution date has passed.
{% endstep %}

{% step %}
### Re-anchor when a wallet retires



```
 1. REQUEST, on Ethereum: the position must be Active. If the source wallet
     is Live, only governance may request (deliberate rotation off a healthy
     wallet). If the source is retiring (MovingFunds or Closing), anyone may
     request. The target must be a different wallet in the Live state, with a
     free slot under its caps. Amount floor: the anchor must be greater than
     the minimum plus one max fee.
  2. SIGN, on Bitcoin: the source wallet's signers check the authorization
     and sign a one-input/one-output re-anchor transaction paying the
     target wallet. When a wallet is moving its funds, signer software
     requests and signs these re-anchors automatically.
  3. PROVE, on Ethereum: an SPV maintainer proves the transaction. The Bridge
     moves custody to the target, shrinks the claim by the fee, releases the
     source wallet's slot, and asks the vault to finance the fee: the vault
     burns tBTC from its reserve, and any shortfall becomes public,
     repayable fee debt.
  4. LATE PROOF: a signed, confirmed re-anchor can be proven with no time
     limit; a late proof can even restore a position that was stranded in
     the meantime onto the new wallet.
  IF THE REQUEST TIMES OUT: anyone can file the timeout. The target slot is
  released, the position returns to Active on the same source wallet, and a
  cooldown applies to the next non-governance request. No one is slashed.
```

Once a retiring wallet holds no reservations and its remaining balance is below the dust threshold, signer software reports that, so the wallet can begin closing.
{% endstep %}

{% step %}
### How a position ends in M1

Stranding is M1's only terminal state. Anyone may file it when the position is `Active` , and the custodying wallet is `Terminated`, `Closed`, or `Closing` with its dissolution date passed. The Bridge releases the capacity, drops the anchor from tracking, and records the position as Stranded. The owner's tBTC balance does not change; what is lost is the in-kind option of getting that specific bitcoin back, which v2 can honor. There is no compensation mechanism: if the bitcoin is really gone, the shortfall affects tBTC backing as a whole, exactly as with any terminated tBTC wallet today.
{% endstep %}
{% endstepper %}
