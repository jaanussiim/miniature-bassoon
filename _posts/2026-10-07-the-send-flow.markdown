---
layout: post
title:  "The Send Flow: an Open Wallet post-mortem"
date:   2026-10-07 05:00:00 +0300
---
To send crypto, on a high level, you just need the recipient address, the token that is sent, and the amount sent.

In Open Wallet, this consisted of three input/output features:
* ChooseCryptoRecipientFeature - **input:** token, **output:** recipient address
* SendCryptoFeature - **input:** token and recipient address, **output:** crypto send confirmation payload with amount
* ConfirmTransactionFeature - **input:** send confirmation payload, **output:** broadcast payload

All these were single-use contained features with a delegate callback, orchestrated by SendCryptoFlowReducer.

In addition to amount entry, SendCryptoFeature was checking the sent token balance and native token balance - can this amount be sent, and is there enough native balance for network fees?

If everything checks out, the user can proceed to ConfirmTransactionFeature. There, you can see overall details of the transfer and initiate the Passkey sign and broadcast action.

At the time, I did not know I had just built the plumbing to send almost anything.

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-send-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-send-input.png' | relative_url }}" alt="Send crypto amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-send-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-send-confirm.png' | relative_url }}" alt="Send crypto confirmation" width="240">
  </a>
</div>

### The DEX
The first swap service added was implemented using [LI.FI][lifi]. This added the additional step of the user choosing the target token, but at a high level it was still just - you have a token and amount to send. The quote contained smart contract data to execute, which plugged into existing send logic nicely.

The input validation logic from token send would also apply here. With both swapped coin and native token balance checks. To reuse existing validation logic, I introduced an abstraction where the original SendCryptoFeature was renamed to TransactionInputFeature. I extracted the crypto send input elements to a new SendCryptoFeature.

{% highlight swift %}
@Reducer
public struct TransactionInput: Sendable {
  @Reducer
  public enum Entry {
    case send(SendCrypto)
    case swap(SwapCrypto)
  }

  @ObservableState
  public struct State: Equatable, Sendable {
    var entry: Entry.State
  }
}
{% endhighlight %}

The same extension was performed on the confirmation screen

{% highlight swift %}
@Reducer
public struct ConfirmTransaction: Sendable {
  @Reducer
  public enum Confirmed {
    case dex(SwapQuotePolling)
    case send
  }
  
  @ObservableState
  public struct State: Equatable, Sendable {
    public internal(set) var confirmed: Confirmed.State
  }
}
{% endhighlight %}

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-swap-dex-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-dex-input.png' | relative_url }}" alt="DEX swap amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-swap-dex-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-dex-confirm.png' | relative_url }}" alt="DEX swap confirmation" width="240">
  </a>
</div>

### The CEX
For the next one, we added swap using [Changelly][changelly]. Having one swap service implemented, adding a second one was trivial on the input level. On a swap service client side, we retrieved service-specific target token data that was mapped to the common token format used in the app, to be presented in the UI. 

An additional step with Changelly was that after the user had proceeded from amount entry, on the confirmation step, we first needed to retrieve the Changelly quote to execute. The received quote contained the final crypto address where to send crypto.

First, extend the used swap services list, used in the SwapCrypto feature

{% highlight swift %}
public enum SwapService: Sendable, Equatable {
  public enum Decentralized: Sendable, Equatable {
    case lifi
  }

  public enum Centralized: String, Sendable, Equatable, Encodable {
    case changelly
  }

  case dex(Decentralized)
  case cex(Centralized)
}
{% endhighlight %}

And in the confirm step
{% highlight swift %}
@Reducer
public enum Confirmed {
  ...
  case cex(CEXTransactionConfirmation)
  ...
}
{% endhighlight %}

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-swap-cex-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-cex-input.png' | relative_url }}" alt="CEX swap amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-swap-cex-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-swap-cex-confirm.png' | relative_url }}" alt="CEX swap confirmation" width="240">
  </a>
</div>

### Send NFT
Sending an NFT does not have the 'amount entry' state. You choose the recipient and go through the same transaction confirmation action with a custom preview component.

Extending the confirm step
{% highlight swift %}
@Reducer
public enum Confirmed {
  ...
  case nftTransfer(ConfirmNFTTransfer)
  ...
}
{% endhighlight %}

We experimented with an Open Wallet Origins collection. Owners of the NFT would get airdrops to cover the network fees. I specifically requested a rose gold one to be created and that I also receive it. Because the rose gold phones were the prettiest ones. And the pink iPhone 16 I currently own is the best one from recent years.


<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-nft-confirm.jpg' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-nft-confirm.jpg' | relative_url }}" alt="NFT send confirmation" width="240">
  </a>
</div>

### Offramp
For offramp, we used [DTR][dtr]. There, you have a local bank IBAN associated with a stablecoin. When selling the coin, we first fetch the associated crypto address for offramp. From here on, we are back in the familiar enter input, confirm, and broadcast flow. In addition, if you squint, it is kind of a swap operation - swap coin for fiat. To facilitate this in the UI, I introduced hardcoded fiat-specific pseudo tokens, allowing us to present the conversion UI.

I extended the services list used in the swap input component
{% highlight swift %}
public enum SwapService: Sendable, Equatable {
  ...
  case dtrOfframp
}
{% endhighlight %}

And added a confirmation component
{% highlight swift %}
@Reducer
public enum Confirmed {
  ...
  case offramp(DTRConfirmOfframp)
  ...
}
{% endhighlight %}

<div style="display: flex; flex-wrap: wrap; gap: 1rem;">
  <a href="{{ '/assets/images/2026/10/ow-offramp-input.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-offramp-input.png' | relative_url }}" alt="Offramp amount input" width="240">
  </a>
  <a href="{{ '/assets/images/2026/10/ow-offramp-confirm.png' | relative_url }}" target="_blank">
    <img src="{{ '/assets/images/2026/10/ow-offramp-confirm.png' | relative_url }}" alt="Offramp confirmation" width="240">
  </a>
</div>


### Experiments
We had backend support for interacting with [Morpho][morpho] vaults. The backend supported creating Morpho vaults and moving money in and out of them. The high-level flow was: choose a market to open a vault on. The vault has a token assigned, enter an amount, ask the backend for broadcast details, execute. Now, looking back, the logic was straightforward, and I never played with taking money out of the vault - there is time for that in the future. I may have left 10 USDC there.

An additional experiment explored was making predictions in the same general send flow. This was backed by [Jupiter predictions][jupiter-predictions] (after a mad one-week dash to switch from another provider). You choose a position to take, enter the amount to send, and execute. On a high level, you were swapping tokens into prediction contracts. We ended up implementing a custom UI flow for that, to make the execution flow faster and avoid the extra confirmation step. All the same methods were executed in the background.

### The C in TCA
In the end, I should have renamed the SendCryptoFlowReducer to SendAnythingFlowReducer. Over the evolution of the feature, small feature-specific interchangeable components were created to make the different variations of send happen. 

We'll never know where else Open Wallet could have gone, but if it had involved sending crypto, I would have first integrated it into the existing send flow. Of course, the glue code and payloads used in the flow could have been made clearer and stricter, but overall it worked out beautifully.

### Epilogue: making the shots
With Open Wallet shutting down, I was faced with a problem: how to take screenshots from an app that has no working backend available? Answer: [Dependencies][point-free-dependencies] and the Live client pattern used. First, remove 37 live dependencies from the SPM build target
{% highlight swift %}
.library(
  name: "ApplicationPackages",
  targets: [
    // "AnalyticsClientLive",
    // "APIClientLive",
    "AppFeature",
    ...
    // "SecureStorageClientLive",
    // "SendBitcoinClientLive",
    ...
  ]
)
{% endhighlight %}

Then run the app to find the unimplemented dependencies used for this demo. I implement simple ones and let Claude Code generate more complex response stubs. In total, around 25 demo overrides were needed to get this demo running. Using this technique, there may be additional demos coming in the future.

[lifi]: https://li.fi/
[changelly]: https://changelly.com/
[morpho]: https://morpho.org/
[dtr]: https://www.dtr.org/
[point-free-dependencies]: https://github.com/pointfreeco/swift-dependencies.git
[jupiter-predictions]: https://jup.ag/prediction
