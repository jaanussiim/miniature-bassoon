---
layout: post
title:  "For PointFree fans: an Open Wallet post-mortem"
date:   2026-09-08 05:00:00 +0300
---
### The Join
At the end of 2023, I received a LinkedIn message from Mitul asking if I would like to work on a self-custodial wallet. The profile picture looked like an AI-generated Bollywood villain - sorry, Mitul :) - and the URL did not go anywhere. I forgot that message.

Luckily, he wrote again mentioning TCA. I was trying to play it cool, but he had me at TCA. The rest is now history.

### Raw numbers
* 992 Swift files
* ~87k lines of code
* 241 SPM packages
* 5,702 commits, of which 5,518 by one engineer ;)
* 1,011 merged PRs
* 15 of 51 dependencies from the PointFree ecosystem (7 declared directly in Package.swift; the rest pulled in transitively).

### PointFree packages pinned directly by Package.swift

[ComposableArchitecture][swift-composable-architecture]

This is where the magic happened, across 117 feature packages. Of course, we also encountered the large path enum issue.

[Parsing][point-free-parsing]

* Parsing deep links
* Two types of crypto address parsing - a thorough one for user entry, lightweight match in JSON decoding
* User entry parsing into ENS name, crypto address, and pasted WalletConnect or deep link URLs 

[Dependencies][point-free-dependencies]

* Using the *name*Client and *name*ClientLive package definition strategy.
* 49 client interfaces with 45 client implementations. Four clients were created before I joined without the live implementation separation.
* 6 dedicated native chain implementations (SendBitcoinClient, SendLitecoinClient, SendSolanaClient, SendTonClient, SendTronClient, SendCosmosClient) plus shared EVM handling for Ethereum/Base/Polygon/Arbitrum/BNB — orchestrated by a TokenSendClient.
* Evolution of SSE client getting replaced by WebSocket implementation with only a live dependency change

[Tagged][swift-tagged]

Making the values of base types a bit tighter.
{% highlight swift %}
public enum ID {
  public enum BitcoinAddress {}
  public enum EVMAddress {}
  public enum LitecoinAddress {}
  public enum ChainID {}
}

public typealias BitcoinAddress = Tagged<ID.BitcoinAddress, String>
public typealias EVMAddress = Tagged<ID.EVMAddress, String>
public typealias LitecoinAddress = Tagged<ID.LitecoinAddress, String>
public typealias ChainID = Tagged<ID.ChainID, Int>
{% endhighlight %}

[Sharing][swift-sharing]

Making core values used app-wide accessible.
{% highlight swift %}
public extension SharedReaderKey where Self == InMemoryKey<User?>.Default {
  static var user: Self {
    Self[.inMemory(User.key), default: nil]
  }
}
{% endhighlight %}


[ConcurrencyExtras][swift-concurrency-extras]

Using LockIsolated to make some in-memory variables concurrency-friendly.
{% highlight swift %}
extension WalletConnectClient: DependencyKey {
  public static var liveValue: WalletConnectClient {
    let setupPerformed = LockIsolated(false)
    
    return WalletConnectClient(
      disconnectAll: {
        guard setupPerformed.value else {
          return
        }
        
        // cleanup logic
      },
      eventsWithSetup: {
      	// WalletConnect setup logic
      	
      	setupPerformed.setValue(true)
      }
    )
  }
}
{% endhighlight %}

[SnapshotTesting][swift-snapshot-testing]

18 \_\_Snapshots\_\_ directories for making sure that the views or elements that are composed based on complex logic did not break by accident. Snapshot record mode is used for fast iteration in both light and dark mode without needing to re-launch the app.

<a href="{{ '/assets/images/point-free-offramp-status.png' | relative_url }}" target="_blank">
  <img src="{{ '/assets/images/point-free-offramp-status.png' | relative_url }}" alt="Offramp status snapshot test states in light and dark mode" width="240">
</a>

### Fin
I think I still know almost nothing about crypto.

[swift-composable-architecture]: https://github.com/pointfreeco/swift-composable-architecture.git
[point-free-parsing]: https://github.com/pointfreeco/swift-parsing.git
[point-free-dependencies]: https://github.com/pointfreeco/swift-dependencies.git
[swift-tagged]: https://github.com/pointfreeco/swift-tagged.git
[swift-sharing]: https://github.com/pointfreeco/swift-sharing.git
[swift-concurrency-extras]: https://github.com/pointfreeco/swift-concurrency-extras.git
[swift-snapshot-testing]: https://github.com/pointfreeco/swift-snapshot-testing.git
