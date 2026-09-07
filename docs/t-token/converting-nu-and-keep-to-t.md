---
description: This page explains how you can swap your NU and KEEP tokens to T token.
---

# Converting NU & KEEP to T

If you still hold KEEP or NU, you can still convert your tokens to **T.** The migration began around the launch of Threshold in 2022. That original migration experience is now legacy infrastructure, and the old upgrade UI is no longer a practical way to complete the conversion.

_However, the underlying smart contracts did not disappear._

The KEEP and NU Vending Machine contracts were designed to remain available indefinitely, which means there is no migration deadline for liquid KEEP or NU tokens. If you still have KEEP or NU sitting in your wallet, you can interact with those contracts directly through **Etherscan.**

It looks more technical than the old UI, but the process itself is relatively straightforward.

{% hint style="info" %}
Before starting the swap, make sure your **KEEP or NU is in an Ethereum (ERC-20) wallet.** This wallet should have a **small amount of ETH for gas fees**, and make sure you are connected to the Ethereum Mainnet.
{% endhint %}

The conversion ratios were fixed when Threshold launched and are based on the respective token supplies rather than their market prices. ([https://docs.threshold.network/t-token](https://docs.threshold.network/t-token))

## Upgrading KEEP/NU to T

{% hint style="info" %}
**The official NU Vending Machine contract is:** [https://etherscan.io/address/0x1cca7e410ee41739792ea0a24e00349dd247680e#writeContract](https://etherscan.io/address/0x1cca7e410ee41739792ea0a24e00349dd247680e#writeContract)
{% endhint %}

{% hint style="info" %}
**The official KEEP Vending Machine contract is:** [https://etherscan.io/address/0xE47c80e8c23f6B4A1aE41c34837a0599D5D16bb0#writeContract](https://etherscan.io/address/0xE47c80e8c23f6B4A1aE41c34837a0599D5D16bb0#writeContract)
{% endhint %}

{% stepper %}
{% step %}
### Approve your KEEP/NU

Before the Vending Machine can convert your KEEP/NU, you first need to give the contract permission to use the KEEP you want to upgrade. Open the KEEP/NU Token Addresses on etherscan, listed as:

* KEEP : `0x85eee30c52b0b379b046fb0f85f4f3dc3009afec`
* NU: `0x4fe83213d56308330ec302a8bd641f1d0113a4cc`

a) Then connect the wallet containing your KEEP/NU.

b) Find the function called: `approve`

c) For `spender`, enter the KEEP/NU Vending Machine address

d) For `amount,` enter how much KEEP/NU you want to convert. One slightly confusing part is that Etherscan expects the amount in the token’s smallest unit. KEEP/ NU has 18 decimals.

_1 KEEP/NU_

_`1000000000000000000`_

_10 KEEP/NU_

_`10000000000000000000`_

_100 KEEP/NU_

_`100000000000000000000`_
{% endstep %}

{% step %}
### Confirm the transaction in your wallet&#x20;

This transaction does not convert your KEEP/NU yet. It simply gives the Vending Machine permission to use the specified amount.&#x20;
{% endstep %}

{% step %}
### Convert your KEEP

Once your approval has been confirmed\
a) Open the [KEEP](https://etherscan.io/address/0xE47c80e8c23f6B4A1aE41c34837a0599D5D16bb0#writeContract)/ [NU](https://etherscan.io/address/0x1cca7e410ee41739792ea0a24e00349dd247680e#writeContract) Vending Machines:

b) Select: _Contract → Write Contract → Connect to Web3_

c) Connect the same wallet and find: _`wrap`_ Under `amount` Enter the amount of KEEP/NU you want to convert, again using 18 decimals.

_For example, for 100 KEEP/NU:_

`100000000000000000000`
{% endstep %}

{% step %}
### Click Write and confirm the transaction in your wallet.

Once the transaction is confirmed, the KEEP/NU will have been converted, and the corresponding T will appear in your wallet.
{% endstep %}
{% endstepper %}

## A few things to remember

There is no deadline to upgrade liquid KEEP or NU into T. The Vending Machine contracts were intentionally designed to remain available to legacy token holders indefinitely.  Remember that this requires two transactions:

**Approve → Wrap → Receive T**

You will therefore need enough ETH to cover gas for both transactions.

Most importantly, do not simply send KEEP or NU to the Vending Machine contract address. Use the `approve` and `wrap` functions described above.

If you are upgrading a significant amount and have never interacted directly with a smart contract through Etherscan before, consider converting a small amount first. Once you have confirmed that the T arrived correctly, you can repeat the process for the remaining balance.
