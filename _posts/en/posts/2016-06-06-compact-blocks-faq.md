---
type: posts
layout: post
lang: en
name: compact-blocks-faq
id: en-compact-blocks-faq
title: Compact Blocks FAQ
permalink: /en/2016/06/06/compact-blocks-faq/
version: 1
---
{% include _toc.html %}

"Compact block relay" is a method of reducing the amount of bandwidth used to propagate new blocks to full nodes.

## Summary

Using simple techniques it is possible to reduce the amount of bandwidth necessary to propagate new blocks to full nodes when they already share much of the same mempool contents. Peers send compact block “sketches” to receiving peers, including the block header, DoS-resistant short txids, and a few full transactions predicted to be absent from their peer’s mempool. The peer can then respond to that message by requesting for the transactions in the block sketch it is missing, or if it already has a completed block constructed from its mempool, send the already-constructed block to its own peers. In ‘high bandwidth mode’ a node asks a few fast block-relaying peers to send new blocks directly without announcement, which trades off some bandwidth for a further reduction in round-trip-time(RTT) latency.

![Compact Blocks diagram](https://raw.githubusercontent.com/bitcoin/bips/master/bip-0152/protocol-flow.png)

##What are some useful benchmarks for this?

A typical full 1MB block announcement with 2,500 transactions can be recovered by the receiving node with a block sketch of about about 15KB, plus overhead for each transaction in the block that is not in their mempool.

When running live experiments in ‘high bandwidth’ mode and having nodes send up to 6 transactions they predict their peer doesn’t have, we can expect to see around 86% of blocks propagate with 0.5RTT. Without sending the predicted missing transactions, this number typically drops, sometimes to a bit less than 50%, requiring another full RTT to get the complete block for most cases. Even without transaction prediction a dramatic reduction in required peak bandwidth is achieved as the block-mempool differential in a vast majority of cases are fewer than 6 transactions. 

## How are expected missing transactions chosen to immediately forward?

To reduce the review surface in the initial implementation only the coinbase transaction will be pre-emptively sent. However in the described experiments earlier nodes would choose in-block transactions it didn’t have in mempool to preemptively send. The reasoning is that without additional information, what you don’t know about is likely what your peers don’t know about. With this basic heuristic a large improvement was seen. Many times the simplest solutions are the best.

## Does this scale Bitcoin?

This feature is intended to save peak block bandwidth for nodes, reducing these spikes which can degrade end-user internet experience. However, the centralization pressures of mining exist in a large part due to latency of block propagation, and compact blocks version 1 is not aimed with that end in mind. 

<iframe width="560" height="315" src="https://www.youtube.com/embed/Y6kibPzbrIc" frameborder="0" allowfullscreen> </iframe>

Instead miners will continue to use the [Fast Relay Network](http://bitcoinrelaynetwork.org/) until a lower-latency or more robust solution is developed.

## But don’t these compactblock schemes reduce latency?

Compared to the standard p2p block relay, yes. However a typical full node doesn’t care much about a few seconds of delay for blocks. In contrast, for a mining full node this means the making a profit or losing money. The Fast Relay Network targets a .5RTT by keeping track of what transactions have been sent to which peers and directly sending block differentials to its peers once a block is found. This compact block proposal can realistically bring most new block announcements down to .5RTT at the cost of some additional bandwidth and RTT in ‘high bandwidth mode’, but a non-negligible amount of blocks will require 1.5RTT.

## Who benefits from compact blocks?

Full node users that desire to relay transactions but have little internet bandwidth. If you simply want to save the most bandwidth possible while still relaying blocks to peers, there is `blocksonly` mode already available starting in Core v0.12, as transaction relay is the most bandwidth costly activity of a full node.

## What is the timeline on coding, testing, reviewing and deploying compact block propagation?

The first version of compact blocks has been assigned BIP152, has a working implementation, and is being actively tested by the developer community. 

- BIP152: <https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki>
- Reference implementation: <https://github.com/TheBlueMatt/bitcoin/tree/udp>

## How can this be adapted for miners?

In order to get miner adoption additional improvements to the compact block scheme must be made. These are two-fold: First, replace TCP transmission of block information with UDP transmission. Second, handle dropped packets by using forward error correction(FEC) codes. 

UDP transmission allows data to be sent by the server and digested by the client as fast as the path allows, without worrying about intermittent dropped packets. A client would rather receive packets out of order to construct the block as fast as possible but TCP does not allow this. In order to deal with the dropped packets, FEC codes will be employed. A FEC code is a method of transforming the data into a redundant code, allowing lossless transmission of the original data as long as any of k-of-n of the packets arrive at its destination, where k is not much larger than the original size of the data.

This setup allows serving peers to send blocks immediately upon receipt, and have clients reconstruct blocks being streamed from multiple peers simultaneously. All of this work will continue to build on the compact blocks work already completed. This is a medium-term extension, and development can be watched here: 

- Reference implelementation: <https://github.com/TheBlueMatt/bitcoin/tree/udp-wip>

## Is this new?

The idea of using BIP37 filteredblocks to more efficiently transmit blocks was proposed a number of  years ago. It was also implemented by Pieter Wuille (sipa) in 2013 but he found the overhead made it slow down the transfer.

{% highlight text %}
[#bitcoin-dev, public log (excerpts)]

[2013-12-27]
09:12 < sipa> TD: i'm working on bip37-based block propagation
[...]
10:27 < BlueMatt> sipa: bip37 doesnt really make sense for block download, no? why do you want the filtered merkle tree instead of just the hash list (since you know you want all txn anyway)
[...]
15:14 < sipa> BlueMatt: the overhead of bip37 for full match is something like 1 bit per transaction, plus maybe 20 bytes per block or so
15:14 < sipa> over just sending the txid list

[2013-12-28]
00:11 < sipa> BlueMatt: i have a ~working bip37 block download branch, but it's buggy and seems to miss blocks and is very slow
00:15 < sipa> BlueMatt: haven't investigated, but my guess is transactions that a peer assumes we have and doesn't send again
00:15 < sipa> while they already have expired from our relay pool
[...]
00:17 < sipa> if we need to ask for missing transactions, there is an extra roundtrip, which makes it likely slower than full block download for many connections
00:18 < BlueMatt> you also cant request missing txn since they are no longer in mempool [...]
00:21 < gmaxwell> sounds like we really do need a protocol extension for this.
[...] 00:23 < sipa> gmaxwell: i don't see how to do it without extra roundtrip
00:23 < BlueMatt> send a list of txn in your mempool (or bloom filter over them or whatever)!
{% endhighlight %}

As was noted in the excerpt, simply extending the protocol to support sending individual transaction hashes for requesting transactions as well as individual transactions in blocks ended up allowing the compact blocks scheme to be much simpler, DoS-resistant, and more efficient.

## Further reading resources

- <https://people.xiph.org/~greg/efficient.block.xfer.txt>
- <https://people.xiph.org/~greg/lowlatency.block.xfer.txt>
- <https://people.xiph.org/~greg/weakblocks.txt>
- <https://people.xiph.org/~greg/mempool_sync_relay.txt>
- <https://en.bitcoin.it/wiki/User:Gmaxwell/block_network_coding>
- <http://diyhpl.us/~bryan/irc/bitcoin/block-propagation-links.2016-05-09.txt>
- <http://diyhpl.us/~bryan/irc/bitcoin/weak-blocks-links.2016-05-09.txt>
- <http://diyhpl.us/~bryan/irc/bitcoin/propagation-links.2016-05-09.txt>

