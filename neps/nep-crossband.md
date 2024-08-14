---
NEP: 0
Title: Cross shard bandwidth limits
Authors: Jan Malinowski <jan.ciolek@nearone.org>
Status: New
DiscussionsTo: https://github.com/nearprotocol/neps/pull/0000
Type: Protocol
Version: 1.0.0
Created: 2024-08-13
LastUpdated: 2024-08-13
---

## Summary

When a chunk is applied it produces outgoing receipts that are sent to other shards. The size of receipts to send could potentially be large and the shard might not be able to transfer all of the receipts before the next block, which could result in missing chunks. The incoming receipts are also included in the `ChunkStateWitness` and we must make sure that its size stays under control. Let's limit how many receipts can be sent out to make sure that the nodes have enough bandwidth to send them in time.
We already implemented a basic version of bandwidth limits in the stateless validation NEP ([NEP-509](https://github.com/near/NEPs/blob/master/neps/nep-0509.md)), but it's very barebones, it severely limits the bandwidth to stay on the safe side. Let's implement a proper solution to support higher cross-shard bandwidth.

## Motivation

#### Why do we need bandwidth limits?

NEAR is a sharded blockchain - every shard is expected to do a limited amount of work at every height. Scaling is mostly achieved by adding more shards. This also means that we cannot expect a shard to send or receive more than X MB of data at every height. Without bandwidth limits some shards might be forced to do a lot of work, more than the shard is capable. Asking the shard to process more work than it can handle would result in delays, missed chunks and possibly even chain stalls. It makes sense to add a limit on how much the shard sends/receives at every height, it's in line with NEAR's design.

We haven't really experienced any serious issues caused by large cross-shard traffic in mainnet, but recent protocol changes have caused the issue to become more a cause for concern:
* With stateless validation all of the incoming receipts are kept inside `ChunkStateWitness`. The protocol is very sensitive to the size of `ChunkStateWitness` - when `ChunkStateWitness` becomes too large the nodes are not able to distribute it in time and there are chunk misses, in extreme cases a shard can even stall. We have to make sure that the size of incoming receipts is limited to avoid witness size issues and attacks.
* The number of incoming receipts to a shard scales linearly with the number of shards - the more shards there are, the more receipts they send. As we add more and more shards the size of incoming receipts will become more of an issue, at some point we'd have to introduce the limits to prevent nodes from getting overloaded by incoming receipts. Without these limits NEAR is not scalable. In the last year the number of shards increased by 50%, we might get to 10 shards by the end of year. We need NEAR to be scalable if we want to keep increasing the number of shards.


#### Why is the current solution inadequate?

There is a rudimentary solution in place, added together with stateless validation in [NEP-509](https://github.com/near/NEPs/blob/master/neps/nep-0509.md) to limit witnes size.
In this solution each shard is usually allowed to send 100KiB (`outgoing_receipts_usual_size_limit`) of receipts to another shard, but there's one special shard that is allowed to send 4.5MiB (`outgoing_receipts_big_size_limit`). The special allowed shard is switched on every height in a round robin fashion. If a shards wants to send less than 100KiB it can just do it, but for larger transfers the sender needs to wait until it's the allowed shard to send the receipts. A node can only send more than 100KiB on its turn. See the PR for a more detailed description of the solution: https://github.com/near/nearcore/pull/11492

This solution was simple enough to be implemented before stateless validation launch, but there is a number of issues with this approach:
* Small throughput - If we take two shards - `1` and `2`, then `1` is able to send at most 5MiB of data to `2` every 6 blocks (assuming 6 shards). That's only 800KiB / height, even though in theory NEAR could support 5MiB / height (assuming that other shards aren't sending much). That's a lot unused throughput that we can't make use of because of the overly restrictive limits. There are some use cases that could make use of higher throughput, e.g NEAR DA, although last I heard NEAR DA was moving to a design that doesn't require a lot of cross-shard bandwidth.
* Hiccups on large receipts - when a receipt is larger than 100KiB it can't be sent until the sender is the allowed shard, which could take up to `num_shards` blocks. The outgoing receipt queue is a FIFO queue, so all the other receipts are stuck behind the large receipt as well. This means that a single large receipt can block all outgoing receipts for a few blocks. This is undesirable because it increases latency for all these receipts and makes DoS attacks easier. The problem will become more pronounced once more shards are added. On current mainnet traffic receipts larger than 100KiB occur about once per 1000 blocks, so it's not that big of an issue, but this could change in the future.
* Missing chunks - when a chunk is missing, the next chunk receives both the receipts aimed at the previous chunk and the new one. If there were a few missing chunks in a row, the next chunk will receive incoming receipts from all of the missing ones. We must make sure that the total size of incoming receipts from all these heights doesn't get too large. The current solution deals with this by marking a shard as congested if there were too many missing chunks in a row. A fully congested shard can't receive any receipts, so after a few heights other shards will stop sending receipts to the shard with missing chunks. The shard will receive receipts from a few heights at most. This makes the problem smaller, but doesn't fully fix it. Receipts from multiple heights could still add up to over 15MB, which is quite large. There are also issues with allowed shard, which is allowed to send receipts to fully congested shards, which could make the problem worse in some corner cases.

The current solution has many deficiencies that could be solved by a better approach.

## Specification

The main source of wasted bandwidth is that assigning bandwidth doesn't take into account the needs of individual shards. When shard `1` needs to send 500KiB and shard `2` needs to send 20KiB the algorithm can assign all of the bandwidth to shard `2` even though it doesn't really need it, it just happened to be the allowed shard at this height. This is wasteful, it would be much better if the algorithm could see how much each shard needs and give to each according to their needs.
This is the general idea behind the new solution: each shard requests bandwidth according to its needs and bandwidth scheduler divides the bandwidth between everone that requested it. The bandwidth scheduler would be able to see that shard `2` needs 500KiB of bandwidth and it'd give it to `2`.
The flow will look like this:
* A chunk is applied and produces outgoing receipts to other shards.
* The shard calculates the current limits and sends as many receipts as it's allowed to.
* The receipts that can't be sent due to limits are buffered (saved to state), they will be sent later.
* The shard calculates how much bandwidth it needs to send the buffered receipts and creates a `BandwidthRequest` with this information (there's one `BandwidthRequest` per target shard).
* The list of `BandwidthRequest` from this shard is included in the chunk header and distributed to other nodes.
* When the next chunk is applied it gathers all the `BandwidthRequests` from chunk headers at the previous height(s) and uses `BandwidthScheduler` to calculate the current bandwidth limits in a deterministic way. The same calculation is performed on all shards and all shards arrive at the same bandwidth limits.
* The chunk is applied and produces outoging receipts, receipts are sent until they hit the limits set by `BandwidthScheduler`.

### `BandwidthRequest`

A shard looks at its queue of buffered receipts to another shard and generates a `BandwidthRequest` which describes how much bandwidth the shard would like to have.
In the simplest version a `BandwidthRequest` could be a single integer containing the total size of buffered receipts.
But there is a problem with this simple representation - it doesn't say anything about the size of individual receipts. Let's say that two shards want to send 4MB of data each to another shard, but the incoming limit is 5MB. Should we assign 2.5MB of bandwidth to each of the sender shards? That would work if the shards want to send a lot of small receipts, but it wouldn't work when each shard wants to send a single 4MB receipt. A shard can't send a part of the 4MB receipt, it's either the whole receipt or nothing. The scheduler should assign 2.5MB/2.5MB of bandwidth when the receipts are small and 4MB/0MB when they're large. The simple version doesn't have enough information for the scheduler to make the right decision, so we'll use a richer representation.

The richer reprenentation is a list of possible bandwidths that the shard would like to receive. When a shard has a lot of small receipts in the queue the list could look like this: [100kB, 200kB, 300kB, ..., 3.9MB, 4MB]. When there's one huge receipt the list would be: [4MB] (receiving e.g. 100kB of bandwidth would be useless, so there's only 4MB in the list of options). Shards are able to tell the scheduler what bandwidth assignments make sense for their requests and the bandwidth scheduler is able to assign one of the sensible possibilities for every request.

Conceptually a `BandwidthRequest` looks like this:
```rust
struct BandwidthRequest {
    /// Requesting bandwidth to this shard
    to_shard: ShardId,
    /// Please grant me one of the options listed here.
    possible_bandwidth_grants: Vec<usize>
}
```

A list of such requests will be included in the chunk header:
```rust
struct ChunkHeader {
    bandwidth_requests: Vec<BandwidthRequest>
}
```

With this representation of `BandwidthRequest` the list of bandwidth requests could take up a lot of space in the chunk header. Luckily it's possible to significantly reduce its size using a better representation.
First we could use `u8` instead `u64` for the `ShardId`, NEAR currently has only 6 shards and it'll take a while to reach 255. There's no need to handle 10**18 shards.
Second we can use a bitmask for the `possible_bandwidth_grants`. The requests don't have to be very precise, 100kB granularity would be sufficient. Assuming that the maximum grant is 4.5MB, we could use 45 bits to represent all posible requests. Having `1` in the `n-th` bit would mean that one of the options is `(n+1) * 100kB`. With these optimizations a single `BandwidthRequest` would be only 7 bytes in size. It's possible to further reduce the size by lowering granulariy and/or using exponential scale. TODO - decide on the exact representation used in the implementation and describe it here.

Sending less than 100KiB of receipts won't require a `BandwidthRequest`. Every shard will be allowed to send this much without asking for permission. On current mainnet traffic the typical size of receipts to send is below 20kB, so on most heights there won't be any bandwidth requests. Bandwidth requests are needed only for exceptionally large transfers. This helps to save space inside chunk headers, we don't have to keep `num_shards**2` requests for every height.

The exact algorithm for generating bandwidth requests will be described later.

### `BandwidthScheduler`

`BandwidthScheduler` is an algorithm which looks at all of the `BandwidthRequests` submitted by shards and grants the bandwidth in a fair way.


[Explain the proposal as if you were teaching it to another developer. This generally means describing the syntax and semantics, naming new concepts, and providing clear examples. The specification needs to include sufficient detail to allow interoperable implementations getting built by following only the provided specification. In cases where it is infeasible to specify all implementation details upfront, broadly describe what they are.]

## Reference Implementation

[This technical section is required for Protocol proposals but optional for other categories. A draft implementation should demonstrate a minimal implementation that assists in understanding or implementing this proposal. Explain the design in sufficient detail that:

* Its interaction with other features is clear.
* Where possible, include a Minimum Viable Interface subsection expressing the required behavior and types in a target programming language. (ie. traits and structs for rust, interfaces and classes for javascript, function signatures and structs for c, etc.)
* It is reasonably clear how the feature would be implemented.
* Corner cases are dissected by example.
* For protocol changes: A link to a draft PR on nearcore that shows how it can be integrated in the current code. It should at least solve the key technical challenges.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.]

## Security Implications

[Explicitly outline any security concerns in relation to the NEP, and potential ways to resolve or mitigate them. At the very least, well-known relevant threats must be covered, e.g. person-in-the-middle, double-spend, XSS, CSRF, etc.]

## Alternatives

[Explain any alternative designs that were considered and the rationale for not choosing them. Why your design is superior?]

## Future possibilities

[Describe any natural extensions and evolutions to the NEP proposal, and how they would impact the project. Use this section as a tool to help fully consider all possible interactions with the project in your proposal. This is also a good place to "dump ideas"; if they are out of scope for the NEP but otherwise related. Note that having something written down in the future-possibilities section is not a reason to accept the current or a future NEP. Such notes should be in the section on motivation or rationale in this or subsequent NEPs. The section merely provides additional information.]

## Consequences

[This section describes the consequences, after applying the decision. All consequences should be summarized here, not just the "positive" ones. Record any concerns raised throughout the NEP discussion.]

### Positive

* p1

### Neutral

* n1

### Negative

* n1

### Backwards Compatibility

[All NEPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. Author must explain a proposes to deal with these incompatibilities. Submissions without a sufficient backwards compatibility treatise may be rejected outright.]

## Unresolved Issues (Optional)

[Explain any issues that warrant further discussion. Considerations

* What parts of the design do you expect to resolve through the NEP process before this gets merged?
* What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
* What related issues do you consider out of scope for this NEP that could be addressed in the future independently of the solution that comes out of this NEP?]

## Changelog

[The changelog section provides historical context for how the NEP developed over time. Initial NEP submission should start with version 1.0.0, and all subsequent NEP extensions must follow [Semantic Versioning](https://semver.org/). Every version should have the benefits and concerns raised during the review. The author does not need to fill out this section for the initial draft. Instead, the assigned reviewers (Subject Matter Experts) should create the first version during the first technical review. After the final public call, the author should then finalize the last version of the decision context.]

### 1.0.0 - Initial Version

> Placeholder for the context about when and who approved this NEP version.

#### Benefits

> List of benefits filled by the Subject Matter Experts while reviewing this version:

* Benefit 1
* Benefit 2

#### Concerns

> Template for Subject Matter Experts review for this version:
> Status: New | Ongoing | Resolved

|   # | Concern | Resolution | Status |
| --: | :------ | :--------- | -----: |
|   1 |         |            |        |
|   2 |         |            |        |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
