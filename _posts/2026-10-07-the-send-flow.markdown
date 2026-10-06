---
layout: post
title:  "The Send Flow: an Open Wallet post-mortem"
date:   2027-10-06 05:00:00 +0300
---
### The Send
For receiving crypto, you don't really need to do much on the app side. Bytes appear on the blockchain, and you are done. Right?

To send crypto, on a high level, you just need the recipient address, the token that is sent, and the amount sent.

In Open Wallet, this consisted of three input/output features:
* ChooseCryptoRecipientFeature - **input:** token, **output:** recipient address
* SendCryptoFeature - **input:** token and recipient address, **output:** crypto send confirmation payload with amount
* ConfirmTransactionFeature - **input:** send confirmation payload, **output:** broadcast payload

All these were single-use contained features with a delegate callback, orchestrated by SendCryptoFlowReducer.

In addition to amount entry, SendCryptoFeature was checking the sent token balance and native token balance. It was performing network fee checks from the backend. It performed user input validation for the actual amount that can be sent and for the native token balance for fees.

If everything checked out, the user can proceed to ConfirmTransactionFeature. There, you could see overall details of the transfer and initiate the Passkey sign and broadcast action.

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-send-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-send-input.png' | relative_url }}" alt="Send crypto amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-send-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-send-confirm.png' | relative_url }}" alt="Send crypto confirmation" width="240">
  </a>
</div>

### The DEX
The first swap service added was implemented using [LI.FI][lifi]. This added the additional step of the user choosing the target token, but at a high level it was still just - you have a token and amount to send. And some extra data.

The input validation logic from token send would also apply here. 

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-swap-dex-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-dex-input.png' | relative_url }}" alt="DEX swap amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-swap-dex-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-dex-confirm.png' | relative_url }}" alt="DEX swap confirmation" width="240">
  </a>
</div>

### The CEX
Having one swap service, adding a second one was trivial. For the next one, we added swap using [Changelly][changelly]. On a swap service client level, you retrieved service-specific target token data that was mapped to the common token format used in the app, to be presented in the UI. An additional step with Changelly was that after the user had confirmed the amount to swap, on the confirmation step, we first needed to retrieve the Changelly quote to execute.


<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-swap-cex-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-cex-input.png' | relative_url }}" alt="CEX swap amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-swap-cex-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-cex-confirm.png' | relative_url }}" alt="CEX swap confirmation" width="240">
  </a>
</div>

### Send NFT
Sending the NFT does not have the 'amount entry' state, but does still go through the same transaction confirmation action with a custom preview component.

A rose gold NFT was requested specially by yours truly.

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-nft-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-nft-confirm.png' | relative_url }}" alt="NFT send confirmation" width="240">
  </a>
</div>

### Offramp
For offramp, we used [DTR][dtr]. There, you have a local bank IBAN associated with a stablecoin. When selling the coin, we first fetch the associated crypto address for offramp. From here on, we are back in the familiar enter input, confirm, and broadcast flow. In addition, if you squint, it is kind of a swap operation - swap coin for fiat. To facilitate this in the UI, I introduced hardcoded fiat-specific pseudo tokens, allowing us to present the conversion UI.

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-offramp-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-offramp-input.png' | relative_url }}" alt="Offramp amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-offramp-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-offramp-confirm.png' | relative_url }}" alt="Offramp confirmation" width="240">
  </a>
</div>

### Experiments
We had backend support for interacting with [Morpho][morpho] vaults. Creating a vault. Adding money to the vault and taking money out of the vault. The high-level flow was: choose a market to open a vault on. The vault has a token assigned, enter an amount, ask the backend for broadcast details, execute. Now, looking back, the logic was straightforward, and I never played with taking money out of the vault; there is time for that in the future. I may have left 10 USDC there.

An additional experiment explored was making predictions in the same general send flow. You choose a position to take, enter the amount to send, and execute. We ended up implementing a custom UI flow for that.

[lifi]: https://li.fi/
[changelly]: https://changelly.com/
[morpho]: https://morpho.org/
[dtr]: https://www.dtr.org/