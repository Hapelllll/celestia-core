# ADR 004: Data Availability Sampling light-client

## Changelog

- 2021-05-03: Initial Draft

## Context

We decided to augment the existing [RCP-based Tendermint light client](https://github.com/tendermint/tendermint/blob/bc643b19c48495077e0394d3e21e1d2a52c99548/light/doc.go#L2-L126) by adding the possibility to additionally validate blocks by doing Data Availability Sampling (DAS).
In general, DAS gives light clients assurance that the data behind the block header they validated is actually available in the network and hence, that state fraud proofs could be generated.
See [ADR 002](adr-002-ipld-da-sampling.md) for more context on DAS.

A great introduction on the Tendermint light client (and light clients in general) can be found in this series of [blog posts](https://medium.com/tendermint/everything-you-need-to-know-about-the-tendermint-light-client-f80d03856f98) as well as this [paper](https://arxiv.org/abs/2010.07031).

This ADR describes the changes necessary to augment the existing Tendermint light client implementation with Data Availability Sampling from a UX as well as from a protocol perspective.

## Alternative Approaches

Ideally, the light client should not just request [signed headers](https://github.com/tendermint/tendermint/blob/bc643b19c48495077e0394d3e21e1d2a52c99548/light/doc.go#L35-L52) from [a few pre-configured peers](https://github.com/tendermint/tendermint/blob/bc643b19c48495077e0394d3e21e1d2a52c99548/light/setup.go#L51-L52) but instead also discover peers from a p2p network.
We will eventually implement this. For more context, we refer to this [issue](https://github.com/lazyledger/lazyledger-core/issues/86).
This would require that the (signed) headers are provided via other means than the RPC.
See this [abandoned pull request](https://github.com/tendermint/tendermint/pull/4508) and [issue](https://github.com/tendermint/tendermint/issues/4456) in the Tendermint repository and also this [suggestion](https://github.com/lazyledger/lazyledger-core/issues/86#issuecomment-831182564) by [@Wondertan](https://github.com/Wondertan) in this repository.

For some use-cases - like DAS light validator nodes, or the light clients of a Data Availability that are run by full nodes of an Optimistic Rollup - it would even make sense that the light client (passively) participates in the consensus protocol to some extent; i.e. runs a subset of the consensus reactor to Consensus messages ([Votes](https://github.com/tendermint/tendermint/blob/bc643b19c48495077e0394d3e21e1d2a52c99548/types/vote.go#L48-L59) etc.) come in as early as possible.
Then light clients would not need to wait for the canonical commit to be included in the next [block](https://github.com/tendermint/tendermint/blob/bc643b19c48495077e0394d3e21e1d2a52c99548/types/block.go#L48).

For the RPC-based light client it could also make sense to add a new RPC endpoint to tendermint for clients to retrieve the DA header, or embed the DAHeader.
The [Commit](https://github.com/lazyledger/lazyledger-core/blob/cbf1f1a4a0472373289a9834b0d33e0918237b7f/rpc/core/routes.go#L25) only contains the [SignedHeader](https://github.com/lazyledger/lazyledger-core/blob/cbf1f1a4a0472373289a9834b0d33e0918237b7f/rpc/core/types/responses.go#L32-L36) (Header and Commit signatures).
Not all light clients will need the full DAHeader though (e.g. super-light-client do not).


## Decision

For our MVP, we [decide](https://github.com/lazyledger/lazyledger-core/issues/307) to only modify the existing RPC-endpoint based light client.
This is mostly because we want to ship our MVP as quickly as possible but independently of this it makes sense to provide a familiar experience for engineers coming from the Cosmos ecosystem.

We will later implement the above mentioned variants.
How exactly will be described in separate ADRs though.

## Detailed Design

From a user perspective very little changes:
the existing light client command gets an additional flag that indicates whether to run DAS or not.
Additionally, the light client operator can decide the number of successful samples to make to deem the block available (and hence valid).

In case DAS is enabled, the light client will need to:
1. retrieve the DAHeader corresponding to the data root in the Header
2. request a predefined number of samples.

If the number of samples succeed, the whole block is available (with some high enough probability).

### UX

The main change to the light client [command](https://github.com/lazyledger/lazyledger-core/blob/master/cmd/tendermint/commands/light.go#L32-L104) is to add in a new flag to indicate if it should run DAS or not.

```diff
Index: cmd/tendermint/commands/light.go
===================================================================
diff --git a/cmd/tendermint/commands/light.go b/cmd/tendermint/commands/light.go
--- a/cmd/tendermint/commands/light.go	(revision cbf1f1a4a0472373289a9834b0d33e0918237b7f)
+++ b/cmd/tendermint/commands/light.go	(date 1620077232436)
@@ -64,6 +64,7 @@
 	dir                string
 	maxOpenConnections int

+	daSampling	   bool
 	sequential     bool
 	trustingPeriod time.Duration
 	trustedHeight  int64
@@ -101,6 +102,9 @@
 	LightCmd.Flags().BoolVar(&sequential, "sequential", false,
 		"sequential verification. Verify all headers sequentially as opposed to using skipping verification",
 	)
+	LightCmd.Flags().BoolVar(&daSampling, "da-sampling", false,
+		"data availability sampling. For each verified header, additionally verify data availability via data availability sampling",
+	)
 }

 func runProxy(cmd *cobra.Command, args []string) error {
```

For the Data Availability sampling, the light client will have to run an IPFS node.
It makes sense to make this mostly opaque to the user as everything around IPFS can be [configured](https://github.com/ipfs/go-ipfs/blob/d6322f485af222e319c893eeac51c44a9859e901/docs/config.md) in the $IPFS_PATH.

TODO: align init'ing the ipfs repo for light client and full nodes


### Light Client Protocol with DAS

> This section does not need to be filled in at the start of the ADR, but must be completed prior to the merging of the implementation.
>
> Here are some common questions that get answered as part of the detailed design:
>
> - What are the user requirements?
>
> - What systems will be affected?
>
> - What new data structures are needed, what data structures will be changed?
>
> - What new APIs will be needed, what APIs will be changed?
>
> - What are the efficiency considerations (time/space)?
>
> - What are the expected access patterns (load/throughput)?
>
> - Are there any logging, monitoring or observability needs?
>
> - Are there any security considerations?
>
> - Are there any privacy considerations?
>
> - How will the changes be tested?
>
> - If the change is large, how will the changes be broken up for ease of review?
>
> - Will these changes require a breaking (major) release?
>
> - Does this change require coordination with the LazyLedger fork of the SDK or lazyledger-app?

## Status

Proposed

## Consequences

### Positive

- simple to implement and understand
- familiar to tendermint / Cosmos devs
- allows trying out the MVP without relying on the [lazyledger-app](https://github.com/lazyledger/lazyledger-app) (instead a simple abci app like a modified KV-store app could be used to demo the light client)
- from the RPC serving node a DAS and a non-DAS client look the same

### Negative

- light client does not discover peers
- requires the light client that currently runs simple RPC requests only to run an IPFS node
- even though DAS light clients will need the DAHeader anyways, we require them to do somewhat superfluous network round-trips to download the DAHeader

### Neutral

DAS light clients need to additionally obtain the DAHeader from the data root in the header to be able to actually do DAS.

## References

We have linked all references above inside the text already.