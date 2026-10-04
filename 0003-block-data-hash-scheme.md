---
layout: default
title: Block Data Hash Scheme
nav_order: 3
---

- Feature Name: block_data_hash_scheme
- Start Date: 2026-10-04
- RFC PR: -
- Fabric-X Component: fabric-x-common, fabric-x-orderer, fabric-x-committer
- Fabric-X Issue: -

# Summary
[summary]: #summary

This RFC proposes two related changes to how the block `Header.DataHash` is computed in Fabric-X.

1. **A new, parallelizable two-layer hashing scheme for the block data hash.** The block's data items
   are split into fixed groups of 128. Each group is hashed with today's length-prefixed SHA-256 chain,
   and all groups are hashed in parallel. The resulting group digests are then hashed once more to
   produce the root. The new scheme becomes the **default** for new Fabric-X networks.
2. **A named, bootstrap-time selection of the block data hashing scheme in the channel config.** The
   existing `BlockDataHashingStructure` channel value gets a new `Name` field. The scheme is fixed in
   the genesis block and cannot change afterwards. Today's flat scheme stays available for networks that
   want it, and future schemes (for example a Merkle tree) can be added under new names.

# Motivation
[motivation]: #motivation

Today the block data hash is a single sequential SHA-256 chain over a length-prefixed serialization of
the block's data items:

```
DataHash = SHA256( Σᵢ uint32_be(len(dᵢ)) ‖ dᵢ )
```

It is implemented in fabric-x-common as `protoutil.ComputeBlockDataHash`. The orderer has a
byte-identical twin, `BatchedRequests.Digest()`. The length prefix makes the serialization unambiguous,
and that is what gives the hash its second-pre-image resistance.

The hash is a chain: each `Write` depends on the previous internal hash state. So it **cannot use more
than one core**, and its cost grows linearly with the total bytes in a block. In Arma this value sits on
several hot paths:

- **Batcher.** The primary computes the digest of every batch it creates. It is the signed content of
  the Batch Attestation Fragment (BAF) and later becomes the block `DataHash`. Every secondary
  recomputes it when it pulls the batch and must match it byte for byte before it acks.
- **Consenter.** The consenter computes it for the blocks it produces (decisions and config blocks) and
  for block verification during synchronization.
- **Assembler.** The assembler recomputes it to verify every batch it fetches from a batcher before
  writing the batch to the block ledger.
- **Committer and deliver clients.** These recompute it to verify that each delivered block's header
  matches its data (for example `utils/deliverorderer/verify.go` in fabric-x-committer and
  `deliverclient/block_verification.go` in fabric-x-common).

A typical Arma batch carries about 10,000 transactions. Arma is built to scale throughput horizontally,
so a per-batch computation bound to a single core works against the design. Several roles also run it
more than once per batch.

Benchmark at 10,000 requests × 300 B on an Intel Xeon (Cascadelake), 8 physical cores / 16 threads,
`linux/amd64`:

| scheme                          | ns/op         | vs flat   | B/op      | allocs/op |
|---------------------------------|---------------|-----------|-----------|-----------|
| flat (today, sequential)        | 8,364,314     | 1.0×      | 164       | 3         |
| binary Merkle root (parallel)   | 3,927,403     | 2.1×      | 1,172,885 | 300       |
| **two-layer, K=128 (parallel)** | **1,501,691** | **5.6×**  | 5,990     | 70        |

The reference implementations and benchmarks are on the
[`two-layer-block-data-hash`](https://github.com/tock-ibm/fabric-x-orderer/tree/two-layer-block-data-hash)
branch (`common/types/chunked_digest.go`, `common/types/merkle_digest.go` and their tests).

Any parallel scheme produces a **different value** from the flat chain, so every component that
computes or checks `DataHash` must agree on which scheme is in use. That is why the scheme belongs in
the channel config (Goal 2) and not in code or in local node configuration.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Named block data hashing schemes

A Fabric-X channel declares exactly one *block data hashing scheme* by name. This RFC defines two
names. The final names are open to discussion.

| name             | description                                                          |
|------------------|----------------------------------------------------------------------|
| `SHA256_2L_K128` | **Default.** Two-layer chunked SHA-256 with a group size of K = 128. |
| `SHA256_FLAT`    | The current Fabric-X length-prefixed sequential SHA-256 (see [Drawbacks](#drawbacks) for how it differs from classic Fabric). |

Each name stands for one fully specified algorithm and has no tunable parameters. A future variant
gets a **new unique name**, for example a different group size, a different hash function or a Merkle
tree. Two nodes that agree on the name therefore always compute the same hash.

## The two-layer scheme

```
 data items:   d₀ d₁ … d₁₂₇ │ d₁₂₈ … d₂₅₅ │ … │ d₍M₋₁₎ₖ … d_{N−1}
               └─ group 0 ─┘ └─ group 1 ─┘     └── group M−1 ──┘    (K = 128 items per group;
                     │             │                   │              the last may be shorter)
 level 1        g₀ = H(…)     g₁ = H(…)    …   g_{M−1} = H(…)        H = today's flat, length-prefixed
 (parallel)          │             │                   │              SHA-256 chain, per group
                     └─────────────┴─────────┬─────────┘
 level 2                     root = SHA256( g₀ ‖ g₁ ‖ … ‖ g_{M−1} )   (single hash, 32·M bytes)
                                             │
                                    Header.DataHash
```

Level 1 does almost all the work, and its M groups are independent, so they are hashed on all available
cores. Level 2 hashes only 32 bytes per group: about 2.5 KB for a 10,000-transaction block. It is cheap
even on one goroutine.

## How developers should think about it

The block data hash is no longer one fixed formula. Code that builds or verifies a block must **never
hard-code** the formula. It should get the scheme from the channel config that governs the block, then
call the scheme-aware helper in fabric-x-common, for example:

```go
scheme := channelConfig.BlockDataHashingScheme()   // e.g. "SHA256_2L_K128"
dataHash, err := protoutil.ComputeBlockDataHash(block.Data, scheme)
```

In the orderer, the batch digest is the future block `DataHash`. It is computed with the same scheme by
the batcher primary, the secondaries and the assembler.

## How operators should think about it

New networks get the two-layer scheme without any action. To keep the original flat scheme, the
operator selects it when generating the genesis block, for example in a configtxgen profile:

```yaml
Profiles:
  SampleFabricX:
    BlockDataHashingStructure:
      Name: SHA256_FLAT      # omit to get the default, SHA256_2L_K128
    Orderer:
      ...
```

The same option is exposed in the armageddon shared-config template.

The scheme is a bootstrap-time decision. A config update that tries to change it is rejected, for
example:

```
error validating channel config update: BlockDataHashingStructure.Name cannot be changed
after bootstrap (current: SHA256_2L_K128, proposed: SHA256_FLAT)
```

An unknown name in a genesis block or config update is also rejected:

```
unknown block data hashing scheme: SHA256_2L_K64
```

## Migration and compatibility

- **No live migration.** The scheme of an existing network cannot change. Fabric-X networks are being
  bootstrapped fresh, so this RFC does not gate the new scheme behind a channel capability.
- **Existing genesis blocks keep their meaning.** A `BlockDataHashingStructure` without the new `Name`
  field (or with an empty one) is read as `SHA256_FLAT`. A network that was bootstrapped before this
  change therefore keeps verifying correctly after its nodes upgrade.
- **External tools** that recompute `DataHash` must read the scheme from the config, or they will
  report mismatches on two-layer networks. Examples are block explorers, the Fabric-X SDK
  ([RFC 0002](0002-client-sdk.md)) and custom verifiers.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Two-layer scheme (`SHA256_2L_K128`) specification

Inputs: the block's data items `d₀ … d_{N−1}` (`BlockData.Data`) in block order, and the constant
`K = 128`.

```
M      = ⌈N / K⌉
group i  = items [ i·K , min((i+1)·K, N) )                 for i = 0 … M−1
gᵢ     = SHA256( Σ_{j ∈ group i}  uint32_be(len(dⱼ)) ‖ dⱼ )   (32 bytes)
root   = SHA256( g₀ ‖ g₁ ‖ … ‖ g_{M−1} )                     (input: 32·M bytes)
DataHash = root
```

Edge cases:

- **N = 0** (empty or nil data): `M = 0`, so `root = SHA256(ε)`. This is the same value the flat scheme
  produces for empty data.
- **1 ≤ N ≤ K**: `M = 1`, so `root = SHA256(g₀)`. There is deliberately **no shortcut** that returns `g₀`
  directly (see [Security](#security-collision-and-second-pre-image-resistance)). As a result, a
  single-group block has a different `DataHash` than the same block under `SHA256_FLAT`.
- Groups are formed by **item count only**, never by byte size, so the grouping is a deterministic
  function of N.

Reference implementation (adapted from `common/types/chunked_digest.go` on the branch linked above):

```go
const chunkGroupSize = 128 // K, fixed by the scheme name SHA256_2L_K128

func TwoLayerDataHash(items [][]byte) []byte {
	n := len(items)
	if n == 0 {
		return sha256.New().Sum(nil)
	}
	numGroups := (n + chunkGroupSize - 1) / chunkGroupSize

	// Level 1: one digest per group, written into disjoint slots of a shared array.
	groupDigests := make([]byte, numGroups*sha256.Size)
	forEachRangeInParallel(numGroups, func(start, end int) {
		h := sha256.New()           // one hasher per worker, reused via Reset
		sizeBuff := make([]byte, 4) // one length buffer per worker
		for g := start; g < end; g++ {
			lo, hi := g*chunkGroupSize, min((g+1)*chunkGroupSize, n)
			h.Reset()
			for _, d := range items[lo:hi] {
				binary.BigEndian.PutUint32(sizeBuff, uint32(len(d)))
				h.Write(sizeBuff)
				h.Write(d)
			}
			h.Sum(groupDigests[g*sha256.Size : g*sha256.Size : (g+1)*sha256.Size])
		}
	})

	// Level 2: a single hash over the concatenated group digests.
	root := sha256.Sum256(groupDigests)
	return root[:]
}
```

A set of normative test vectors will be checked into fabric-x-common during implementation. They map
fixed inputs (N = 0, 1, K−1, K, K+1, 2K, 10,000, including empty items) to expected roots. Every
implementation must reproduce them.

## Security: collision and second-pre-image resistance

**Threat.** Given block data `D = (d₀ … d_{N−1})`, an adversary wants to find different block data `D′`
such that `DataHash(D′) = DataHash(D)` (a *second pre-image*). For example, a Byzantine batcher could
try this to swap a batch's contents after its BAF was signed. A weaker goal is any pair `D ≠ D′` with
equal hashes (a *collision*). The following argument shows that either attack requires breaking SHA-256
itself.

1. **Level 1 is injective on its input.** Within a group, every item is preceded by its 4-byte length.
   Reading the serialized byte string from left to right recovers the exact item list: read a length,
   read that many bytes, repeat. So two different item lists can never produce the same serialized
   string, and `gᵢ = gᵢ′` for different groups is a SHA-256 collision. This is the same property the
   flat scheme relies on today. It rules out the classic boundary-shifting attack, where `[ab, cd]` and
   `[a, bcd]` concatenate to the same bytes.
2. **Level 2 has a fixed-width, unambiguous input.** The root hashes `g₀ ‖ … ‖ g_{M−1}`, and every `gᵢ`
   is exactly 32 bytes. The input's length is `32·M`, which determines M, and its split into digests is
   unique. So `root(D) = root(D′)` with a different digest sequence is a SHA-256 collision. Otherwise
   M = M′ and `gᵢ = gᵢ′` for every i.
3. **The grouping is not attacker-controlled.** K is fixed by the scheme name and is not stored in the
   block or chosen by the producer. Given M, every group except the last holds exactly K items. The last
   group's size is bound by its own injective length-prefixed serialization (point 1). So equal digest
   sequences imply the same items in the same order, which means `D = D′`.
4. **No interior-node-as-leaf confusion.** The classic second-pre-image attack on Merkle trees presents
   an interior node's input as if it were leaf data, or exploits odd-node duplication (Bitcoin,
   CVE-2012-2459). RFC 6962 prevents it with `0x00` / `0x01` domain-separation prefixes. Here the
   structure has a **fixed depth of two**, so such confusion is not possible:
   - A level-2 input is never interpreted as level-1 data, because every block has exactly one level-2
     hash and it is always the root.
   - Nothing is duplicated.
   - A single-group block still applies the level-2 hash (`root = SHA256(g₀)`), so a root is never a
     bare group digest. If the `M = 1` shortcut were allowed, an attacker could look for item lists
     whose length-prefixed serialization equals some `g₀ ‖ g₁ ‖ …`. Avoiding the shortcut removes that
     question entirely.

Breaking the scheme therefore requires a SHA-256 collision (or a second pre-image) at one of the two
levels. Explicit domain-separation prefixes are **not required**. They could be added as defense in
depth in a future named scheme.

## Performance characteristics

- **Work.** The scheme performs `M + 1` SHA-256 finalizations, for example 80 for N = 10,000. It
  streams the same bytes as the flat scheme plus 32·M bytes at level 2, so its total work is barely
  above the flat scheme, and that work is spread across cores. A binary Merkle tree by comparison
  performs about `2N` finalizations, each with its own padding block.
- **Implementation notes.** Use one reused `sha256` hasher per worker (`sha256.Sum256` allocates a
  hasher per call on Go's FIPS path). Write group digests into disjoint slots of one shared array with
  no locking. Fan out as soon as `M ≥ 2`.
- **Crossover.** The two-layer scheme is faster than flat as soon as `N > K` (two or more groups). For
  `N ≤ K` it costs one extra 32-byte hash, which is negligible.

  ns/op at 300 B per item:

  | N      | flat       | Merkle    | two-layer K=128 |
  |--------|------------|-----------|-----------------|
  | 100    | 82,131     | 144,297   | 86,021          |
  | 500    | 407,689    | 722,881   | **160,401**     |
  | 1,000  | 861,436    | 917,046   | **302,704**     |
  | 4,000  | 3,272,922  | 2,242,708 | **706,866**     |
  | 16,000 | 13,869,129 | 5,872,197 | **2,427,933**   |

- **Choice of K = 128.** A sweep of K ∈ {16 … 2048} over N ∈ {1,000 … 10,000} shows a flat optimum for
  K ∈ [64, 256]. K = 128 is fastest or within noise almost everywhere. Large K (≥ 512) leaves cores idle
  once `M` drops below the core count, and K = 1024 at N = 1,000 is about 3× slower. The rule of thumb is
  `M ≥ cores`, i.e. `K ≤ N / cores`.
- **Scaling.** SHA-256 is ALU-bound, so speedup is bounded by **physical** cores. Hyperthreads do not
  help. On machines with more cores, the advantage grows further.

## Configuration (Goal 2)

### Proto

The Fabric channel-level config value `common.BlockDataHashingStructure` already exists in every
Fabric-X genesis block. It is written by configtxgen (`encoder.go`, via
`channelconfig.BlockDataHashingStructureValue()`) under the channel group with `mod_policy: Admins`.
Today it holds only `width`, which Fabric reserved for a future tree-hashing structure and which is
validated to be `MaxUint32` only (`channelconfig/channel.go: validateBlockDataHashingStructure`). This
RFC adds a name field:

```protobuf
// BlockDataHashingStructure encapsulates information about the block data hashing structure.
message BlockDataHashingStructure {
    // width is kept for compatibility; Fabric-X requires MaxUint32 and otherwise ignores it.
    uint32 width = 1;

    // name selects the block data hashing scheme, e.g. "SHA256_2L_K128" or "SHA256_FLAT".
    // An empty name means "SHA256_FLAT".
    string name = 2;
}
```

### fabric-x-common `channelconfig`

- `validateBlockDataHashingStructure` keeps requiring `width == MaxUint32`. It also requires `name` to
  be empty or one of the registered scheme names, and rejects anything else.
- A new accessor `ChannelConfig.BlockDataHashingScheme()` returns the scheme name, mapping an empty name
  to `SHA256_FLAT`. It is exposed through the `channelconfig.Channel` interface next to the existing
  `BlockDataHashingStructureWidth()`.
- **Immutability.** When a config update is validated, a change to the effective scheme name is
  rejected. The comparison uses the normalized name, so empty and `SHA256_FLAT` are equal.

### fabric-x-common `protoutil`

- A small registry maps each scheme name to a hasher. `ComputeBlockDataHash` / `BlockDataHash` take the
  scheme (or a resolved hasher) as a parameter. The current flat function remains as the implementation
  of `SHA256_FLAT`.

### configtxgen and armageddon

- The configtxgen `Profile` gets an optional `BlockDataHashingStructure: { Name: … }` section. When it
  is omitted, `BlockDataHashingStructureValue()` writes `name: SHA256_2L_K128`. This is what makes the
  new scheme the default for newly generated genesis blocks.
- `armageddon generate` and its shared-config template expose the same option.

### Out of scope

The channel's `HashingAlgorithm` value (SHA256 / SHA3_256) is left unchanged. `DataHash` remains
SHA-256-based under both schemes. A SHA3 variant would be a new scheme name.

## Which blocks use which scheme

- **Genesis block.** Its `DataHash` is computed with the scheme declared **inside** it. A verifier
  bootstrapping from a genesis block first parses the config envelope, reads the scheme, then verifies
  the header.
- **All later blocks** (data and config) use the same scheme, because the scheme is immutable. A
  verifier never needs to switch schemes mid-chain.
- **Batches and BAFs.** The batch digest that a batcher signs in a BAF equals the future block
  `DataHash`. It is computed with the channel's scheme by the primary, the secondaries, the consenters
  (when they build blocks) and the assembler (when it verifies fetched batches).
- **Consenter-internal blocks.** These are the blocks of the consenter's own SmartBFT ledger. They follow
  the same scheme, for uniformity. See
  [Unresolved questions](#unresolved-questions).

## Impact per component

This is the starting point for the implementation plans in each repository.

- **Protos**: add `name` to `BlockDataHashingStructure` (`common/configuration.proto`), then regenerate.
- **fabric-x-common**:
  - the `protoutil` scheme registry and scheme-aware `ComputeBlockDataHash` / `BlockDataHash`.
  - `channelconfig` validation, the accessor and the immutability check.
  - the configtxgen profile and default.
  - callers: `common/genesis/genesis.go`, `common/deliverclient/block_verification.go`,
    `common/ledger/blockledger/util.go`, test utilities.
- **fabric-x-orderer**:
  - batcher: `BatchedRequests.Digest()` becomes scheme-aware, covering the primary's BAF creation and the
    secondaries' verification.
  - assembler: `node/assembler/batch_fetcher.go`.
  - consensus: `CreateDataCommonBlock` / `CreateConfigCommonBlock` in `node/consensus/utils.go`,
    `node/consensus/consensus.go`, `node/consensus/state/decision.go`, `consensus_builder.go` and
    `synchronizer/consenter_block_verifier.go`.
  - shared: `node/comm/util.go`, `common/deliverclient`, `armageddon generate`.
  - The scheme is obtained from the shared config / config block at bootstrap.
- **fabric-x-committer**: `utils/deliverorderer/verify.go`, any other `DataHash` checks in the sidecar
  and query service, and `loadgen/workload/mappers.go`.

**Phasing.**

1. protos
2. fabric-x-common, with both schemes and a scheme-aware API, default still flat
3. fabric-x-orderer and fabric-x-committer become scheme-aware, verifying both schemes
4. switch the configtxgen / armageddon default to `SHA256_2L_K128`

Each step can merge on its own without breaking existing networks.

# Drawbacks
[drawbacks]: #drawbacks

- **Divergence from classic Fabric, under either scheme.** Fabric-X already differs from Hyperledger
  Fabric here, even with `SHA256_FLAT`. Classic Fabric hashes the plain concatenation of the data
  items, with no length prefixes:

  ```go
  // hyperledger/fabric protoutil/blockutils.go
  func ComputeBlockDataHash(b *cb.BlockData) []byte {
      sum := sha256.Sum256(bytes.Join(b.Data, nil))
      return sum[:]
  }
  ```

  Taken on its own, this formula is ambiguous: `[ab, cd]` and `[a, bcd]` hash to the same value, so
  item boundaries can be shifted without changing `DataHash`. Fabric relies on a separate check that
  every item is a well-formed envelope (`VerifyTransactionsAreWellFormed`), not on the hash itself.
  Fabric-X fixed this by adding the 4-byte length prefixes. So Fabric-X `SHA256_FLAT` blocks already
  fail verification in tools that hard-code the Fabric formula. The two-layer scheme does not create a
  new incompatibility with classic Fabric. It adds a second Fabric-X scheme that block explorers, SDKs
  and auditors must recognize. Because the default changes, this applies to every new network that does
  not opt into `SHA256_FLAT`.
- **One more config dimension.** Mismatched implementations (for example an old committer) show up as
  `DataHash` mismatches on every block, not as a clear "unsupported scheme" error, unless the verifier
  checks the scheme name first. Implementations should fail fast on an unknown scheme.
- **Small blocks pay one extra hash.** For `N ≤ 128` the scheme costs one additional SHA-256 over 32
  bytes. This is negligible, but non-zero.
- **Coarse inclusion proofs.** This is not a Merkle tree. Proving that a transaction is in a block
  requires its whole group (up to 128 items) plus the other `M − 1` group digests. That is far smaller
  than the whole block, but larger than a logarithmic Merkle path.

# Rationale and alternatives
[alternatives]: #alternatives

**Binary Merkle tree (RFC 6962-style).** A Merkle root over one leaf per item, with `0x00` / `0x01`
domain separation and odd nodes promoted, not duplicated. It was implemented and benchmarked alongside
the two-layer scheme. Compared to the **flat** scheme it is parallel, about 2.1× faster at 10,000 items,
and it gives logarithmic per-transaction inclusion proofs. Compared to the **two-layer** scheme it is
clearly worse:

- **About 2.6× slower** at 10,000 × 300 B (3.93 ms vs 1.50 ms).
- **About 2N vs M + 1 finalizations** (about 20,000 vs 80 at N = 10,000), each with its own padding
  block. Single-threaded it is about 1.7× *slower than flat*, and it only wins by spreading extra work
  across cores.
- **About 200× more memory and 4× more allocations per call** (1.17 MB / 300 allocs vs 6 KB / 70), even
  after hasher reuse and shared backing arrays.
- **Late crossover.** It only overtakes flat at about 1,500–2,000 items, whereas two-layer wins from
  N > 128.
- It plateaus at the physical core count (3.8× at 8 cores, no gain at 16 threads).

Arma's ordering path does not currently need per-transaction inclusion proofs, so the Merkle tree's
extra cost buys nothing today. The named-scheme design means a Merkle scheme (for example
`SHA256_MERKLE`) **can be added later as a new name** with no further config changes, if light clients
or proofs become a requirement.

**Reuse `Width` as the group size K.** `BlockDataHashingStructure.width` was originally intended to
parameterize a hashing tree, so it is tempting to read `width = 128` as "two-layer, K = 128". This was
rejected for three reasons:

- It overloads a field with existing Fabric semantics.
- It allows arbitrary K values, which multiplies the space of supported and tested variants.
- A single integer cannot name schemes that are not chunked trees, such as Merkle or SHA3 variants.

An explicit name, with one fully specified algorithm per name, keeps the set of supported schemes small
and auditable.

**Gate behind a channel capability.** Fabric gates protocol changes behind capabilities so that existing
networks can upgrade in place. Fabric-X networks are bootstrapped fresh and the scheme is fixed at
bootstrap, so a capability adds machinery with no benefit now. It can be revisited if in-place migration
is ever needed.

**Local (per-node) configuration.** Rejected. Every node of every role, and every external verifier,
must compute the same value, so the scheme must be part of the agreed channel config.

**Allow changing the scheme in a config update.** Rejected. It would force every verifier to track
scheme changes block by block, and it would complicate batches that straddle a config change. Fixing
the scheme at bootstrap keeps verification stateless with respect to the scheme.

**Hard-code the two-layer scheme with no option.** Simpler, but it would remove the option to keep the
current Fabric-X block format for existing tooling, and it leaves no clean path for future schemes.

**Do nothing.** The block data hash remains a single-core, linear-time computation that several Arma
roles repeat for every batch. That limits throughput as batch sizes and transaction rates grow.

# Prior art
[prior-art]: #prior-art

- **Hyperledger Fabric `BlockDataHashingStructure.width`.** Fabric anticipated configurable block data
  hashing structures but never implemented them. Only `MaxUint32` (the flat hash) is accepted. This RFC
  builds on that existing hook.
- **RFC 6962 (Certificate Transparency).** Merkle hash trees with leaf and interior domain separation.
  This is the basis of the Merkle alternative evaluated here.
- **Bitcoin.** Uses a transaction Merkle root. Its odd-node duplication led to CVE-2012-2459 (block
  malleability), which is a cautionary example for tree-hash design. The two-layer scheme has no
  duplication and a fixed depth.
- **Ethereum.** Commits to transactions with a Merkle-Patricia trie root (`transactionsRoot`), trading
  computation for inclusion proofs.
- **Tree and chunked hashing modes.** BLAKE3 (chunked Merkle tree for parallelism), the Sakura coding and
  KangarooTwelve tree mode for Keccak, and the SHA-3 tree-hashing discussions. They share the same idea:
  hash fixed-size chunks independently, then combine the chunk digests.

# Testing
[testing]: #testing

- **Normative test vectors** in fabric-x-common for each scheme name. They cover N = 0, 1, K−1, K, K+1,
  2K and 10,000, empty items, and large items. fabric-x-orderer and fabric-x-committer tests consume the
  same vectors.
- **Equivalence tests**: the parallel implementation against a single-threaded reference oracle (each
  group computed with the flat function), under `-race`, across many random N and item sizes.
- **Config tests** in `channelconfig` and configtxgen:
  - A generated genesis block defaults to `SHA256_2L_K128`.
  - An empty name is read as `SHA256_FLAT`.
  - Unknown names are rejected.
  - `width ≠ MaxUint32` is rejected.
  - A config update that changes the scheme is rejected.
- **Integration tests** in fabric-x-orderer (`test/basic`, `test/reconfig`):
  - Arma networks bootstrapped with each scheme order and deliver blocks.
  - Batcher secondaries, consenters and assemblers agree on digests.
  - Config updates, including the attempted scheme change, behave correctly.
- **End-to-end tests** with fabric-x-committer: the committer's deliver verifier accepts blocks under
  both schemes and rejects a block whose `DataHash` was computed with the wrong scheme.
- **Benchmarks**: keep the flat / two-layer comparison benchmarks and re-run them on CI-class hardware to
  confirm the speedup.

# Dependencies
[dependencies]: #dependencies

- **fabric-protos**: the new `name` field in `BlockDataHashingStructure`. Alternatively a
  Fabric-X-owned proto (see below).
- **fabric-x-common**: the scheme registry, scheme-aware hashing API, channel config validation and
  configtxgen. Everything below depends on it.
- **fabric-x-orderer** and **fabric-x-committer**: adopt the scheme-aware API and obtain the scheme from
  the config block.
- **[RFC 0002 — Fabric-X SDK](0002-client-sdk.md)**: its block parsers and verifiers should use the
  scheme-aware API.

# Unresolved questions
[unresolved]: #unresolved-questions

To resolve through the RFC process:

- **Final scheme names.** `SHA256_FLAT` / `SHA256_2L_K128`, or another naming convention.
- **Where the new proto field lives.** Upstream `hyperledger/fabric-protos` or a Fabric-X-owned
  extension. The upstream path keeps one message; a Fabric-X proto avoids an upstream dependency.
- **The future of `width`.** Whether to formally deprecate it or keep requiring `MaxUint32` for
  compatibility.

To resolve during implementation:

- Whether consenter-internal ledger blocks must use the channel's scheme or may stay flat. They are
  not served to clients, but uniformity simplifies the code.
- The exact shape of the fabric-x-common API: a scheme-name parameter or a resolved hasher object
  obtained from the channel config.

Out of scope, possible future RFCs:

- Additional named schemes: a Merkle tree for inclusion proofs, SHA3-based variants, or schemes with
  explicit domain separation.
- Migrating an existing network from one scheme to another.
