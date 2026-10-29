# Decoupled Storage Market: Implementation Guide

> Companion to [Decoupled Storage Market](./README.md) (the authoritative specification).
>
> This document contains **implementation-specific details only**: protobuf definitions, store layouts, module and
> provider code changes, the upgrade handler, and the test plan. It does not repeat specification-level content.
> For the design, behavior, rules, and rationale, see [README.md](./README.md).

Target repos: `chain-sdk`, `node`, `provider` (go.work monorepo layout), and `helm-charts` (the storage operator
chart).

Reference implementation: branch `storage-market/active-versions` of
[chalabi2/chain-sdk](https://github.com/chalabi2/chain-sdk/tree/storage-market/active-versions),
[chalabi2/node](https://github.com/chalabi2/node/tree/storage-market/active-versions), and
[chalabi2/provider](https://github.com/chalabi2/provider/tree/storage-market/active-versions), and branch
`storage-market/core` of [chalabi2/helm-charts](https://github.com/chalabi2/helm-charts/tree/storage-market/core).
These branches also carry lease resizing, which the In-Place Lease Resize AEP specifies and this document leaves out.

---

## 1. Codebase-Specific Implementation Notes

### 1.1 Proto Placement: The Active Versions

Every consensus-shape change is an additive field on the API versions the network already runs, plus one new stable
package for the provider data plane:

| Package                           | Additions                                                                    |
| --------------------------------- | ---------------------------------------------------------------------------- |
| `akash.deployment.v1` (stable)    | `volume.proto` (`VolumeRef`, `VolumePolicy`), `EventVolumeAdopted`, errors 27-32 |
| `akash.deployment.v1beta4`        | `GroupSpec.volume = 4`, `ResourceUnit.volumes = 4`, `Params` fields 2-4       |
| `akash.market.v1` (stable)        | volume close reasons, `EventVolumeAttached`/`Detached`, `Lease.closed_at = 8` |
| `akash.market.v1beta5`            | `Params` fields 6-7, `MsgCreateBid.volume_policy = 6`                         |
| `akash.escrow.v1` (stable)        | `EventAccountRunway`                                                          |
| `akash.manifest.v2beta3`          | `StorageParams.volume = 4`                                                    |
| `akash.volume.v1` (new)           | the `VolumeTransfer` gRPC service                                             |

`base.resources.v1beta4` is untouched, so deployed inventory, manifest, and operator surfaces see no new resource
shape. The staged `deployment/v1beta5` and `market/v2beta1` directories are untouched and stay excluded in
`buf.yaml`.

**Old-binary safety is a bid-time gate, not a decode failure.** Volume orders are ordinary `market.v1beta5.Order`s
that every provider decodes; an old binary drops `GroupSpec.volume` as an unknown field. Three gates keep old binaries
from holding a volume:

1. the SDL puts `capabilities/storage/volumes: "true"` (or `capabilities/storage/buckets: "true"`) into the placement
   requirements, which providers without volume support do not advertise;
2. `CreateBid` on a volume order requires `MsgCreateBid.volume_policy` equal to the order's policy (market error 107),
   and an old binary never sets it;
3. an attach order is biddable only by the provider holding the volume's active lease.

**Registration.** The storage market adds no Msg types and no Query RPCs, so there are no new `RegisterInterfaces` or
`RegisterCustomSignerField` entries. (The `codec.go` additions on the reference branches belong to lease resizing.)

### 1.2 Zero New Msg Types; AuthorizeDeposits Untouched

The complete flow rides existing messages (see the message table in the README); `MsgCreateBid` gains a field, not a
type.

- The grant switch (`node/x/escrow/keeper/keeper.go:253-283`) already covers `escrow.v1.MsgAccountDeposit`,
  `deployment.v1beta4.MsgCreateDeployment`, and `market.v1beta5.MsgCreateBid`, and is untouched.
- **Test obligation:** grant-funded volume deployments (`node/tests/e2e/storagemarket_test.go`).

### 1.3 Escrow Hooks Hardening (prerequisite-grade, ships in U1)

`OnEscrowAccountClosed` returns `nil` for any XID that fails `DeploymentIDFromEscrowID`, mirroring the payment hook
(`node/x/market/hooks/hooks.go:34-37`): hook errors used to abort `saveAccount` **mid-write** (refunds and `store.Set`
already applied). This AEP adds no new scope and never triggers the bug, but it corrupts state for any future scope.

**Test obligation:** the foreign-scope hook error is masked by `_ = k.ekeeper.AccountClose(...)` at
`node/x/market/keeper/keeper.go:347`, which swallows it for `ScopeBid`. `keeper_escrow_masking_test.go` and
`hooks_test.go` cover the masking and prove bid-escrow account-close behavior is byte-identical after the fix.

### 1.4 Canonical Empty Manifest

A storage-only deployment has no services. The canonical volume manifest is
`[{"name":"<placement name>","services":null}]`; the CLI hashes it for `MsgCreateDeployment.Hash`. Manifest `v2beta3`
validation accepts zero-service groups, and `CheckAgainstGSpecs` pairs them with the chain: zero services on a
non-volume group is `ErrInvalidManifest`, and services on a volume group is `ErrManifestCrossValidation`. The hash for
placement `us-west`, `74066c155b207484e1e3671548c44e5c9dff4d2fbd1293f61fd4f7f466a24e0e`, is pinned in Go tests
(`go/sdl/v2_2_test.go`, `go/cli/volume_tx_test.go`) -- this hash is consensus-adjacent and gets manifest-hash rigor.

The provider never receives a manifest for a volume lease: its manifest manager rejects an upload for a deployment
whose leases on that provider are all volume leases.

### 1.5 Feature Flag

`market.v1beta5.Params.volume_orders_enabled` (field 6, default `false`, governance-flipped) gates:

- volume-group `CreateDeployment`, checked in `x/deployment` through the `MarketKeeper.VolumeOrdersEnabled` method on
  `dimports.MarketKeeper` (`node/x/deployment/imports/keepers.go`), failing with `ErrVolumeOrdersDisabled`
  (deployment 27);
- defense-in-depth, `CreateBid` on volume orders (market 100).

Compute deployments, bids, and leases that reference volumes, `MsgStartGroup`, and the re-order of a moved volume are
not flag-checked: they need a volume lease, which cannot exist while the flag has always been off. A flag turned off
after launch stops new volumes and volume bids and leaves existing volumes working.

### 1.6 SDL Version Gate

`sdl.UnmarshalYAML` (`chain-sdk/go/sdl/sdl.go`) routes `== 2.0.0 -> v2`, `>= 2.1.0 && < 2.2.0 -> v2_1`,
`>= 2.2.0 && < 3.0.0 -> v2_2`, and `sdl-input.schema.yaml` moves in lockstep: the version enum is `2.0 | 2.1 | 2.2`,
`volumes` is allowed only in 2.2, and a 2.2 file needs services or volumes. v2 and v2.1 reject
`params.storage.*.volume`. `interface SDL` gains `Volumes()`. `Read` and `ReadFile` take `ReadOption`s;
`WithOwner(owner)` supplies the signer for references that omit `external.owner`.

---

## 2. Protobuf Definitions

gogoproto options (equality, custom names, nullability, durations) are omitted below.

### 2.1 `akash/deployment/v1/volume.proto` (stable package)

```proto
// VolumeRef addresses one Storage entry of a storage-only ("volume") group.
// String form: owner/dseq/gseq/name (ParseVolumeRef).
message VolumeRef {
  string owner = 1;   // cosmos.AddressString
  uint64 dseq  = 2;
  uint32 gseq  = 3;   // always 1
  string name  = 4;   // Storage.Name in the volume group
}

// VolumePolicy marks a group as a storage-only volume group and carries its
// lifecycle contract. Presence of this message IS the volume discriminator.
message VolumePolicy {
  enum ReclaimPolicy {
    invalid = 0;   // VolumeReclaimInvalid
    retain  = 1;   // VolumeReclaimRetain
    delete  = 2;   // VolumeReclaimDelete
  }

  // Access declares how deployments reach a volume's data.
  enum Access {
    mount  = 0;    // VolumeAccessMount: a disk compute leases on its provider attach
    bucket = 1;    // VolumeAccessBucket: an S3 bucket served over the network; never attached
  }

  // vid: tenant-chosen data-continuity label, ^[a-z0-9]([-a-z0-9]{0,61}[a-z0-9])?$.
  // Typed field -- NEVER a storage attribute (attributes flow into provider
  // attribute matching and would make the order unbiddable).
  string vid = 1;

  ReclaimPolicy reclaim = 2;

  // retention: post-close window the provider keeps the data.
  // Bounded by deployment Params.max_volume_retention; priced into the bid.
  google.protobuf.Duration retention = 3;

  uint32 max_attachments = 4;   // 1 for mount, 0 for bucket
  uint32 max_replicas = 5;      // export duty declared -> priced into the bid; 0 for bucket

  VolumeRef adopt = 6;          // adoption of a dead retained volume
  VolumeRef replica_of = 7;     // this volume is a replica

  Access access = 8;            // json "access,omitempty"
}
```

`VolumePolicy` lives in `deployment/v1` because it is identity and state vocabulary imported by
`deployment.v1beta4` (group spec, resource unit), `market.v1beta5` (the bid), and the `market.v1` and
`deployment.v1` events. `mount` is `0` and the field is `omitempty`, so the JSON, amino sign bytes, and golden
fixtures of disk volume policies carry no `access` at all.

### 2.2 `akash.deployment.v1beta4`

```proto
message GroupSpec {
  // name = 1, requirements = 2, resources = 3 unchanged
  akash.deployment.v1.VolumePolicy volume = 4;   // nil for every compute group
}

message ResourceUnit {
  // resource = 1 (base.resources.v1beta4.Resources -- UNCHANGED base type),
  // count = 2, price = 3 unchanged
  repeated akash.deployment.v1.VolumeRef volumes = 4;
  // externally leased volumes this unit's services mount; empty for every other unit
}

message Params {
  // field 1 unchanged
  uint64 max_volume_size = 2;                          // bytes; default 32Ti; must be > 0
  google.protobuf.Duration max_volume_retention = 3;   // default 720h
  uint32 max_volume_replicas = 4;                      // default 4
}
```

`market.v1beta5.Order.spec` (field 3) embeds this `GroupSpec`, so the volume policy and attach references flow into
orders -- and therefore into bid engine requests -- with no other order-path proto change.

### 2.3 `akash.market.v1` (stable) -- additive vocabulary only

Identity types (`OrderID`/`BidID`/`LeaseID`, `Reclamation`) are untouched -- the `mv1.LeaseID(bid.ID)` raw struct
cast and the exactly-5-part payment-XID parse stay safe. `Lease` gains one field:

```proto
int64 closed_at = 8;   // block time (unix seconds) of the close; 0 while open or for leases closed before the upgrade
```

Close reasons (`market/v1/types.proto`, established ranges: 1-9999 owner, 10000+ provider, 20000+ network):

```proto
reason_volume_migrate       = 101;     // tenant MsgCloseLease on a volume lease: a move
reason_volume_evict         = 10101;   // provider reclaim that re-lists the volume
reason_volume_closed_retain = 20101;   // tenant closed the volume; retention runs
reason_volume_unfunded      = 20102;   // escrow exhaustion cascade
reason_volume_detach        = 20103;   // compute lease force-closed by cascade-detach
```

Typed events (`market/v1/event.proto`, emitted via `EmitTypedEvent`):

```proto
message EventVolumeAttached {
  akash.market.v1.LeaseID lease_id = 1;          // compute lease
  akash.deployment.v1.VolumeRef volume = 2;
}
message EventVolumeDetached {
  akash.market.v1.LeaseID lease_id = 1;
  akash.deployment.v1.VolumeRef volume = 2;
  akash.market.v1.LeaseClosedReason reason = 3;
}
```

Deployment event (`deployment/v1/event.proto`):

```proto
message EventVolumeAdopted {
  akash.deployment.v1.GroupID id = 1;          // new volume
  akash.deployment.v1.VolumeRef adopted = 2;   // dead volume
  string vid = 3;
}
```

### 2.4 `akash.market.v1beta5`

```proto
message Params {
  // existing fields (incl. Min/MaxReclamationWindow) unchanged
  bool volume_orders_enabled = 6;                              // THE feature flag; default false, gov-flipped
  google.protobuf.Duration min_volume_reclamation_window = 7;  // default 24h
}

message MsgCreateBid {
  // existing fields unchanged
  akash.deployment.v1.VolumePolicy volume_policy = 6;  // must equal the order's on a volume order; unset otherwise
}
```

`Params` fields 8-9 on the reference branches belong to lease resizing.

### 2.5 `akash.escrow.v1` (`event.proto`; the only escrow-package touch; no state change)

```proto
// Emitted when a settle of an account with open payments advances at least one
// block and leaves the account funded: the tenant-computable exhaustion deadline.
message EventAccountRunway {
  akash.escrow.id.v1.Account id = 1;
  int64 projected_depletion_height = 2;   // 0 = funds cover no full block
}
```

### 2.6 Manifest `akash.manifest.v2beta3`

```proto
message StorageParams {
  string name = 1;
  string mount = 2;          // mount topology stays off-chain, as today
  bool read_only = 3;
  // volume mirrors the on-chain VolumeRef as "owner/dseq/gseq/name";
  // empty, and omitted from JSON, for locally provisioned (AEP-15) storage.
  string volume = 4;
}
```

`Service.checkVolumesAgainstResources` requires a service's manifest volume references to equal its unit's
`ResourceUnit.volumes` exactly, compared after sorting by the full `owner/dseq/gseq/name` string. Because the field is
omitted when empty, every existing manifest hashes exactly as before; the TS manifest hasher mirrors the omission
(`generateManifestVersion.ts`).

### 2.7 `akash.volume.v1.VolumeTransfer` (data plane, off-chain)

Served by the provider daemon on its gateway gRPC port. `Export` and `ExportObjects` are server streams (gRPC only);
`Status` also declares an HTTP mapping (`GET /v1/volume/status`) that the provider does not serve.

```proto
service VolumeTransfer {
  rpc Export(ExportRequest) returns (stream ExportChunk);               // a disk: full image or diff
  rpc ExportObjects(ExportObjectsRequest) returns (stream ObjectChunk); // a bucket: every object
  rpc Status(StatusRequest) returns (StatusResponse);
}

message ExportRequest {
  string owner = 1;
  string vid = 2;
  uint64 dseq = 3;               // the volume deployment on the EXPORTER's side
  string requester = 4;          // destination provider address (must match the mTLS client cert)
  string from_snapshot = 5;      // "" = full; else the diff base
  uint64 resume_offset = 6;
}
message ExportChunk { bytes data = 1; bytes sha256 = 2; uint64 offset = 3; }

message ExportObjectsRequest {
  string owner = 1;
  string vid = 2;
  uint64 dseq = 3;
  string requester = 4;
  string start_after = 5;        // resume after the last object that landed
}
message ObjectAttributes {
  uint64 size = 1;
  string content_type = 2;
  string content_encoding = 3;
  string content_disposition = 4;
  string content_language = 5;
  string cache_control = 6;
  string expires = 7;
  map<string, string> metadata = 8;   // x-amz-meta-*
}
message ObjectChunk {
  string key = 1;
  ObjectAttributes attributes = 2;    // first chunk of an object only
  uint64 offset = 3;
  bytes data = 4;
  bytes sha256 = 5;                   // this chunk
  bytes object_sha256 = 6;            // last chunk of an object: the whole object
}

message StatusRequest { string owner = 1; string vid = 2; uint64 dseq = 3; string requester = 4; }
message StatusResponse {
  repeated string snapshots = 1;
  string latest_snapshot = 2;
  uint64 size = 3;
  bytes sha256 = 4;                   // whole-image digest
  int64 sync_lag_seconds = 5;
  bool final = 6;                     // the source's lease is closed and nothing mounts the volume
}
```

Both transfer errors are `FailedPrecondition` with ErrorInfo domain `akash.network`: `VOLUME_IMPORT_INCOMPLETE` (the
exporter's own copy never finished arriving; the destination tries the next-older holder) and `VOLUME_NOT_FINAL`
(`ExportObjects` on a bucket that is not final yet; the destination asks again next round).

### 2.8 Error Codes

`deployment/v1` (chain-sdk):

| Code | Error                      | gRPC code          |
| ---- | -------------------------- | ------------------ |
| 27   | `ErrVolumeOrdersDisabled`  | FailedPrecondition |
| 28   | `ErrVidInUse`              | AlreadyExists      |
| 29   | `ErrVidRetained`           | FailedPrecondition |
| 30   | `ErrVolumeAttached`        | FailedPrecondition |
| 31   | `ErrInvalidVolumeAdoption` | InvalidArgument    |
| 32   | `ErrInvalidVolumeReplica`  | InvalidArgument    |

Governance-bound failures use the existing `ErrInvalidParam` (22).

`market` (node, `x/market/handler/errors.go`):

| Code | Error                                | Meaning                                                  |
| ---- | ------------------------------------ | -------------------------------------------------------- |
| 100  | `ErrVolumeOrdersDisabled`            | volume orders disabled                                   |
| 101  | `ErrVolumeReclamationWindowTooShort` | bid window below `min_volume_reclamation_window`         |
| 102  | `ErrVolumeOfferMismatch`             | storage offer does not equal the order's quantities      |
| 103  | `ErrVolumeAdoptionProvider`          | adoption bids are gated to the retaining provider        |
| 104  | `ErrVolumeRefInvalid`                | invalid volume reference                                 |
| 105  | `ErrVolumeNotColocated`              | the volume has no active lease with this provider        |
| 106  | `ErrVolumeAttachmentLimit`           | attachment limit reached                                 |
| 107  | `ErrVolumePolicyNotAccepted`         | the bid's volume policy does not match the order         |
| 108  | `ErrVolumeNotAttachable`             | the volume is a bucket                                   |

A repeated tenant close before the reclamation deadline reuses `ErrReclamationWindowNotElapsed` (38).

---

## 3. Store Layout (no new store keys; indexes inside existing modules)

```
x/deployment store (existing key):
  {0x13,0x01}  pending-denom map (existing)
  {0x13,0x02}  volumes_by_vid         Map[Pair[owner, vid] -> GroupID]                  (NEW)
               written on every volume create, adoption included; never removed:
               whether the volume is live, retained, or expired is read from its group
  {0x13,0x03}  volume_replica_counts  replica_of backrefs per primary volume            (NEW)

x/market store (existing key; orders 0x11, bids 0x12, leases 0x13, params 0x14):
  {0x15,0x00}  attachments     KeySet[Pair[(volume owner, dseq, gseq), compute LeasePrimaryKey]]  (NEW)
  {0x16,0x00}  attachmentsRev  KeySet[Pair[compute LeasePrimaryKey, (volume owner, dseq, gseq)]]  (NEW)
               written and pruned ONLY inside the keeper (never handlers), so every close
               path -- tenant, provider, escrow cascade -- maintains them; iteration is
               deterministic (key order)
```

The indexes are not genesis fields: `InitGenesis` rebuilds them from groups and leases (`x/deployment/genesis.go`,
`x/market/genesis.go`), and the upgrade handler rebuilds them the same way. No module changes its `ConsensusVersion`
(deployment 8, market 9, escrow 3), no store migration is registered, and `StoreLoader.Added` stays empty: no
store-key-mount-on-exact-upgrade-name hazard.

---

## 4. Chain Module Changes (`node`)

No new module, no new stores, no new hooks, no EndBlocker anywhere. `app/` and every `module.go` are untouched.

### 4.1 chain-sdk `deployment/v1beta4` Go Validation (ships in U1, inert behind the flag)

`groupspec.go`, `resourceunit.go`, `group_validation.go`, `msgs.go`:

- `GroupSpec.Volume != nil` (volume groups): one resource unit with ID > 0 and no endpoints; CPU/Memory/GPU
  **present-but-zero** (fields non-nil, values zero -- every downstream dereference stays crash-safe); exactly one
  named Storage entry with `persistent=true`, a `class` attribute other than `ram`, size in `[5Mi, 32Ti]`, and
  `count == 1`; `ResourceUnit.Volumes` empty (a volume never references one); `vid` grammar; `reclaim` is `retain` or
  `delete`; `access` is `mount` or `bucket`; retention >= 0. A disk has `max_attachments == 1`; a bucket has
  `max_attachments == 0`, `max_replicas == 0`, and no `replica_of`. `adopt` and `replica_of` are mutually exclusive
  and point at `gseq == 1`.
- Governance bounds are a separate call, `GroupSpec.ValidateVolumeBounds(Params)` (size <= `max_volume_size`,
  retention <= `max_volume_retention`, `max_replicas` <= `max_volume_replicas`), which the deployment handler runs.
- `GroupSpec.Volume == nil` (compute groups): existing rules unchanged, plus each `VolumeRef` stateless-validated
  (`gseq == 1`, unique within the unit). **No Storage entry shares a reference's name** -- attached volumes contribute
  zero storage quantity, never double-counted in compute inventory or pricing.
- `ValidateDeploymentGroups`: a deployment containing a volume group contains **only** that group (single-group
  invariant => `gseq` always 1; escrow account == volume). `MsgCreateDeployment.ValidateBasic` also requires the
  `adopt` and `replica_of` owners to equal the message owner.

### 4.2 `x/deployment` (`node/x/deployment/handler/server.go`, `keeper/volume.go`)

`CreateDeployment`, after existing validation, when the single group is a volume group:

1. Feature flag: reject unless `VolumeOrdersEnabled` (27).
2. Governance bounds: `ValidateVolumeBounds(params)` (22).
3. **vid gate:** look up `volumes_by_vid[owner, vid]`. A holder group that is open or paused => `ErrVidInUse` (28). A
   holder that is closed or `insufficient_funds` and still within its retention window, counted from the `closed_at`
   of its most recently closed lease => `ErrVidRetained` (29) unless `VolumePolicy.adopt` names exactly that volume.
   Otherwise a fresh create, and the entry is (over)written.
4. **Adoption** (`adopt` set): the referenced group exists, is a volume group of the same owner, is closed or
   `insufficient_funds`, has the same `vid` and `access`, has a storage entry named as the reference, is no larger
   than the new volume, and block time is within its retention window; else `ErrInvalidVolumeAdoption` (31). The
   keeper emits `EventVolumeAdopted`.
5. **Replica** (`replica_of` set): the target exists and is a live disk volume group (not a bucket) of the same owner,
   with a storage entry named as the reference, no larger than the new volume, and with a replica count below its
   `max_replicas`; else `ErrInvalidVolumeReplica` (32). The count drops when the replica closes.
6. Proceed exactly as today: `market.CreateOrder(...)` then the escrow account. The deployment's `Reclamation` field
   stays optional; the floor is enforced on the bid.

`CloseDeployment`, `CloseGroup`, and `PauseGroup` on a volume: rejected with `ErrVolumeAttached` (30) while the
attachment index has entries for it -- tenants close compute first. Otherwise the volume lease enters reclamation with
`volume_closed_retain`. The escrow cascade cannot be blocked; it cascade-detaches instead (4.4).

### 4.3 `x/market` (`node/x/market/handler/server.go`, `keeper/keeper.go`, `keeper/volume.go`)

`msgServer.CreateBid` on a volume order, in this order:

1. flag (100);
2. `msg.VolumePolicy` equals the order's policy (107);
3. `msg.ReclamationWindow >= params.MinVolumeReclamationWindow` (101);
4. `ResourcesOffer` storage quantities **equal** the order's per-name quantities (102);
5. adoption orders: `msg.ID.Provider` equals the provider of the dead volume's most recently closed lease, ordered by
   close height then oseq -- pure chain state (103).

A bid that carries a volume policy on a non-volume order is rejected. Capability matching rides
`order.MatchResourcesRequirements` unchanged (the SDL injects the capability attribute into the placement
requirements).

Attaching-group bids (any `ResourceUnit.Volumes` non-empty), per reference: the group exists, is a volume group of
the order's owner, and has a storage entry of the referenced name (104); it is not a bucket (108); it has a lease in
state `Active` whose provider is the bidder, so a volume in reclamation refuses new attaches (105) -- colocation reads
*current* state, so it survives the re-order loop and migration for free; its attachment count is below
`max_attachments` (106).

`msgServer.CreateLease` re-runs every attach check; the keeper then writes both index directions and emits
`EventVolumeAttached`.

**Shared keeper close routine** (`keeper.go`), reached by `MsgCloseLease`, `MsgCloseBid`, the escrow hooks
(`OnEscrowPaymentClosed`, `OnGroupClosed`), and `CloseGroup`/`PauseGroup`, at the single point where
`EventLeaseClosed` is emitted:

- Closing lease in `attachmentsRev` (a compute attach): prune both directions, emit `EventVolumeDetached{reason}`.
- Closing lease is a volume lease: cascade-detach, then reason substitution (4.4).

**Tenant volume close routes through reclamation.** `MsgCloseLease` on a volume lease calls `ReclaimVolumeOnClose`
with `volume_migrate`: the first close enters `LeaseReclaiming` with
`deadline = now + max(bid.ReclamationWindow, params.MinVolumeReclamationWindow)` and closes nothing; a close after
the deadline completes it and the group re-orders; a close before the deadline fails with
`ErrReclamationWindowNotElapsed` (38). The deployment and group closes of 4.2 route the same way with
`volume_closed_retain`.

**Re-order carve-out:** provider reclaim with `volume_evict` or `volume_migrate` keeps the existing still-open-group
re-order loop (it *is* the move's re-listing). Any other provider reclaim reason on a volume pauses the group without a
re-order, and the lease records that reason. Terminal closes (`volume_unfunded` => group `insufficient_funds`;
`volume_closed_retain` => group closed) fire no re-order -- adoption takes over.

### 4.4 Cascade-Detach (keeper close routine, every path)

Invariant: a volume lease never reaches `closed` while the attachment index holds entries for its group.

1. Reclaim completion (provider `MsgCloseBid` after the deadline; tenant close after its window): iterate the
   volume's attachments in deterministic key order, force-close each attached compute lease with
   `reason_volume_detach` (normal compute close path per lease: payment closed, event emitted, its group re-orders),
   then close the volume lease with the substituted volume reason.
2. Escrow exhaustion: payments are saved before the account, so `OnEscrowPaymentClosed` reaches the routine first; it
   runs the same cascade-detach inline, then closes with `reason_volume_unfunded`. Deterministic, single-tx.
3. A final assert-empty on the volume's attachment entries guards the invariant (`keeper/volume.go`).

While a compute lease is attached, the tenant's only close path is `MsgCloseLease` (a move); deployment, group, and
pause closes are rejected. Bounded: at most `max_attachments` (= 1 in v1) extra closes per volume -- O(1),
gas-accounted.

### 4.5 Escrow (`node/x/escrow/keeper/keeper.go`)

- `EventAccountRunway` is emitted at the end of `accountSettle` when the settle covers at least one new block and
  leaves the account funded: `projected_depletion_height = current + floor(Funds[0] / sum(open payment rates))`. No
  state change; generic across scopes.
- Preserved by not touching: the payments-before-account save ordering; `AccountDeposit`/overdraw semantics; AEP-75
  FIFO drain and grant re-credit; single-denom uact settlement.

---

## 5. Provider Changes

### 5.1 StorageClasses (`helm-charts`, `akash-storage-operator`)

The chart installs akash-labeled `beta1-retain`, `beta2-retain`, and `beta3-retain` StorageClasses (RBD,
`reclaimPolicy: Retain`, `helm.sh/resource-policy: keep`), a node labeler that appends them to the
`akash.network/storageclasses` node label, and the `akash-volumes` namespace. **Invariant: every PV backing a volume
group is created from a retain class.** The AEP-15 per-lease path keeps its Delete classes untouched. The provider
offers the classes listed in `--volume-classes`, which must name classes with a `-retain` variant; the operator picks
the class by name and does not re-check its reclaim policy.

### 5.2 Storage Operator (`provider/operator/storage/`)

A cobra subcommand, `provider-services operator storage` -- the CRD-reconciler archetype (hostname/ip operator
model). One replica; all state re-derivable from CRDs, PVs, and chain queries; nothing only-in-memory.

**Volume CRD** `volumes.akash.network` (group `akash.network`, version `v2beta2`, kind `Volume`, short name `vol`),
namespaced in `--k8s-manifest-ns` (default `lease`). Name: `volume-<sha256(owner/vid)[:12]>`; every lookup by name
also checks owner and vid. Spec: owner, vid, groupId, class, size, reclaim, retention, leaseId, access, importFrom.
Status: `phase`, `pvName`, `attachedLease`, `retainedUntil`, `localCopy`, and for buckets `bucket`, `bucketDSeq`,
`bucketSize`. Phases: `Pending`, `Importing`, `Provisioned` (parked), `Attached`, `Exporting`, `Adopting`, `Retained`,
`Releasing`.

**PV choreography -- never through `Available`** (PVCs are namespaced; PVs are cluster-scoped; the volume's durable
kube object is the PV):

- *Provision*: holder PVC `<crd-name>` in `--volumes-namespace` (default `akash-volumes`) against the retain class;
  the provisioner binds a new PV to it. PV labeled `akash.network/volume-owner|vid|dseq`. Phase `Provisioned`.
- *Attach*: create the target PVC in the compute lease namespace **first** (with `spec.volumeName`), patch
  `pv.spec.claimRef` **directly to the target PVC** (name, namespace, UID), then delete the holder PVC. The PV is never
  an unclaimed `Available` Retain PV the kube binder could hand to an arbitrary pending PVC -- the cross-tenant re-bind
  race is closed by construction.
- *Detach*: when `status.attachedLease` is cleared -- by the daemon's `DetachVolume` at lease teardown, by the compute
  lease's `EventLeaseClosed` or `EventVolumeDetached`, or by a resync that finds the lease inactive -- recreate the
  holder PVC, patch `claimRef` back to it, then delete the stale attach PVC. Phase `Provisioned`.
- *Retain*: stamp `retainedUntil` = the time the operator observed the close + retention (it does not read the
  lease's `closed_at`, so its deadline falls at or after the chain's). *Adopt*: verify chain state (the adopting
  deployment's `VolumePolicy.adopt` matches this volume), update `spec.groupId/leaseId`, re-park. A released PV is
  never adopted for a different owner.
- *GC*: only from `Retained` or `Releasing`, once `retainedUntil` has passed: flip the PV to `Delete`, delete the PVC
  and PV (the RBD image goes with it), then the CRD. A `reclaim: delete` volume goes to `Releasing` at close with
  `retainedUntil = now`. **The single destruction path** -- never directly on `EventLeaseClosed`.

**Chain watch:** its own CometBFT `NewBlockHeader` subscription plus block results, decoding transaction and
finalize-block events: `EventLeaseClosed` (all reasons), `EventLeaseReclaimStarted` (`volume_evict` and
`volume_migrate` only), `EventVolumeAttached`/`Detached`, and `EventVolumeAdopted`. A resync walks this provider's
active leases at start and every `--resync-interval` (default 5m), with no saved cursor and nothing shared with the
bid engine's order discovery.
Without chain access the operator runs degraded: adoptions freeze and re-listed volumes stay `Pending`.

**Daemon wiring:** the daemon reaches the operator's REST API (`/health`, `/state`, `/volume/{owner}/{vid}`) at
`--storage-operator-endpoint` and waits for it at startup only when the provider takes part in the volume market (the
capability attribute, volume classes, or a bucket backend), so non-participants gain no chart-before-daemon ordering
constraint. Other operator flags: `--provision-node-hint` (annotate provisioning claims with the selected node, for
node-constrained provisioners) and `--node` (chain RPC). Helm chart cloned from `akash-inventory-operator` with
matching discovery labels; RBAC covers PVs, PVCs, and `volumes.akash.network`.

**Inventory truth:** (1) the inventory operator publishes volume allocations per class into its merged cluster state,
counting every phase except `Releasing` (`operator/inventory/volumes.go`), so parked, retained, and moving volumes are
never double-counted as free; (2) it rebuilds them from Volume CRDs on restart; (3) the storage operator reports
retained bytes on its `/state` endpoint.

### 5.3 Cluster Layer (`provider/cluster`, `cluster/kube`)

- `cluster.Client` (`cluster/client.go`) gains `DeployVolume`, `AttachVolume`, `DetachVolume`, `TeardownVolume`,
  `VolumeStatus`, `LeaseVolume`, and `DeployedVolumes` (nullClient stubs included); implementations go through the
  Volume CRD -- the daemon never manipulates PVs directly. `DeployVolume` on a re-listed order resets the record to
  `Pending`.
- The daemon drives attach: `AttachVolume` runs before `Deploy` and refuses a volume attached to another lease or
  still importing.
- `Deploy` renders a service with a volume reference as a k8s `Deployment` (not a StatefulSet -- positional
  per-replica VolumeClaimTemplates don't apply), `replicas=1`, mounting the attach PVC by claim name.
- `TeardownLease` is **unchanged** -- namespace deletion removes only the attach PVC; the Retain PV survives and the
  operator re-parks it. Deploy-failure rollback is equally harmless.
- Volume objects carry `akash.network/component: volume` labels excluded from `cleanupStaleResources`' selector.
- Reservations: `ctypes.Reservation` gains a **durable** kind, counted by `inventory.Adjust` per class, with commit
  levels skipped for the whole volume group (durable leased bytes are never thin-sold), exempt from
  unreserve-on-lease-close, and rebuilt on restart from Volume CRDs. A released volume's durable reservation is dropped
  at the next restart; until then the inventory operator's allocation (5.2) carries the capacity truth.
- Manifests: the manifest manager rejects an upload for a deployment whose leases on this provider are all volume
  leases, and never sends one to a volume lease.

### 5.4 Bid Engine (`provider/bidengine`)

- **Discovery: zero changes.** Volume orders are ordinary `EventOrderCreated`/`OrderOpen` on `market.v1beta5`;
  `order.run` forks on `GroupSpec.Volume != nil`.
- **Volume gates** (`volume.go`): placement, audit, and capability checks; the class in `--volume-classes` (or
  `--bucket-classes` for a bucket, which also needs a bucket backend); size <= `--volume-max-size`; retention <=
  `--volume-max-retention`; `max_replicas` <= `--volume-max-replicas`, and a `replica_of` order needs a nonzero cap.
  Both caps default to 0, which declines any order with retention or replicas. The engine waits up to two minutes for
  the first inventory snapshot, then requires live per-class headroom. Adoption orders: a local pre-check for a `Retained` Volume of the same owner and vid, or one
  already `Adopting` or `Provisioned` for the same dseq (the chain gate is the authority).
- **Bids** carry the order's `VolumePolicy` and the provider-wide `--reclamation-window`, which must be at least the
  chain floor for volume bids to land. Bids on a re-listed order (a move or an eviction) stay open for
  `--volume-move-bid-timeout` (default 168h), so the tenant has time to choose the next provider.
- **Pricing:** the shell-script strategy's `storageElement` gains `volume`, `retention_hours`, `max_replicas`, and
  `replica`; scale pricing prices a volume's storage by class. A bucket is priced as a volume of its class.
- **Win handoff:** a `VolumeLeaseWon` bus event reaches the volume provisioner, which creates the Volume CRD and adds
  the lease to the withdrawal loop -- **no manifest wait**; the GroupSpec is the whole contract. `checkForExistingBid`
  restart recovery applies unchanged.
- **Attach orders:** the ordinary compute path plus a local check that each reference names a `Provisioned`,
  unattached Volume of the order's owner and the reference's name as vid; attached storage contributes zero quantity --
  no reservation double-count.

### 5.5 Transfer Service, Moves, and Replication

**Service.** The provider daemon serves `VolumeTransfer` on its gateway gRPC port (`--volume-transfer-port`, default
8444) whenever it takes part in the volume market, over mutual TLS 1.3 with x/cert provider certificates; a request's
`requester` must equal the client certificate's CN. A client pins the server certificate to the source's on-chain
identity and dials the host of the source's host URI at the transfer port, unless `--volume-transfer-endpoints` names
another address for that provider.

**Authorization** (`operator/storage/replication/authz.go`), derived from chain state on both ends: the exporter
serves a requester holding an active lease on the same owner, vid, and dseq, or the provider of a replica whose
`replica_of` names this volume. The importer accepts a source that held a lease on a volume of the same owner and vid
(active, reclaiming, closed, or unfunded), starting with the latest provider whose lease closed with `volume_migrate`
or `volume_evict`.

**Final.** A source is final when its own latest lease on the volume is closed, no pod mounts the claim, and no
VolumeAttachment remains; `Status.final` reports it.

**Driver `stream-export`** (default, `--volume-replication-driver`):

- RBD-backed PVs: `rbd snap create` (`@akash-xfer-<UTC>`), then `rbd export` or `rbd export-diff`, run in the
  rook-ceph-tools pod (`--volume-replication-rook-namespace`, default `rook-ceph`); import is `rbd rm` then
  `rbd import`.
- Other PVs (e.g. local-path): a tar of the volume's host-path tree, staged under `--volume-replication-dir`; always a
  full copy.
- Integrity: 1 MiB chunks, each with its sha256; the image is checked against the whole-image digest; transfers
  resume from the last offset (up to 5 attempts); an import is capped at 1.5 x size + 64 MiB.

`rbd-mirror` is a registered driver name without an implementation; selecting it fails provider startup.

**A move, as run:**

1. Reclaim starts (`EventLeaseReclaimStarted` with a move reason): the source Volume goes to `Exporting`; its workload
   keeps running.
2. At the deadline the provider daemon's volume reclaimer (a 30-second sweep) closes the lease with `MsgCloseBid`,
   unless the tenant's own close got there first. The chain cascade-detaches and re-lists the order. The source stays
   `Exporting` until the observed close + retention, then goes `Retained`, then is released.
3. The winner's operator records `importFrom` from chain and provisions the volume as `Importing`. Its daemon pulls
   every `--volume-migration-interval` (default 1m) until the source reports final, copies the final image or objects,
   and parks the volume as `Provisioned`.
4. A source whose own copy never finished arriving answers `VOLUME_IMPORT_INCOMPLETE`, and the destination tries the
   next-older holder. After about 30 minutes of failed rounds, the destination starts a `volume_evict` reclaim on its
   own lease, and the order re-lists again.

The source's GC freeze ends at `retainedUntil` whether or not a copy finished.

**Replication.** The export authorization serves replica providers, and the replica sync loop (`RunReplica`: an
initial full export, then periodic diffs every `--volume-replica-sync-interval`, default 15m) is implemented, but the
daemon does not start it yet (see §9).

### 5.6 Buckets (`provider/bucket`, storage operator)

**Backend.** A `bucket.Backend` interface (`Ensure`, `Suspend`, `Resume`, `Delete`, `Seal`, `Objects`, key create,
list, and revoke, `Usage`) keeps the S3 server pluggable; v1 ships Ceph RGW through its Admin Ops API. Providers with
persistent storage already run Rook-Ceph, so enabling buckets is adding an object store. A conformance suite runs
against an in-memory fake and, under the `rgw_integration` build tag, a live RGW.

**Mapping onto RGW**, for the Volume CRD `volume-<sha256(owner/vid)[:12]>`:

| RGW object          | Value                                                                                   |
| ------------------- | --------------------------------------------------------------------------------------- |
| user uid            | the CRD name -- stable across leases, so adoption keeps the data                        |
| user display name   | `owner/vid`, checked on every call, so a colliding uid never reaches another volume     |
| user quota          | `max-size` = the leased size; writes past it fail                                       |
| user `max-buckets`  | 1 while the provider creates the bucket, then -1 (no key can create a bucket)           |
| bucket              | `<vid, cut to 46>-<hex16(sha256(owner/vid))>`, owned by the user                        |
| bucket policy       | deny `s3:DeleteBucket` to everyone (the admin API that GC uses is not subject to it)    |
| public access block | `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`                        |
| tenant keys         | sub-users with `read` or `readwrite` access (never `full`), at most 20 per bucket       |

The provider creates and guards the bucket with a key it drops before returning, so it keeps no key to tenant data.

**Lifecycle:**

| Event                    | Bucket                                                                                   |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| lease won                | `Ensure` (user, quota, bucket, guard); `Provisioned`                                     |
| reclamation window       | keeps serving (the lease is still paid)                                                  |
| lease closed (no move)   | `Suspend` (every key stops); `Retained`, or `Releasing` for `delete`                     |
| lease closed (move)      | sealed on every pass: every sub-user revoked, the user left unsuspended so the provider can still read it; `Exporting` until close + retention |
| adoption                 | waits for the adopting lease, then revokes every old key and resumes                     |
| `retainedUntil` passed   | delete the bucket, its objects, and the user                                             |

**Moves:** `ExportObjects` serves a bucket only once it is final (its lease closed and no key left), else
`VOLUME_NOT_FINAL`. The destination provisions the bucket as `Importing`, refuses new keys (409) until the copy lands,
empties its own bucket once on the first chunk (it may hold an older copy of its own), spools each object to disk,
verifies every chunk and the whole object, and writes it with its attributes (objects past 256 MiB as 64 MiB multipart
parts, aborted on any failure). A broken stream resumes after the last object that landed. A provider that held the
bucket before and wins it back revokes its earlier keys, lifts the suspension, and imports the newer objects. Not
carried: keys, object tags, ACLs, versions, and incomplete multipart uploads.

**Configuration:** `--bucket-backend rgw` enables buckets, and the provider advertises `capabilities/storage/buckets`
in its on-chain attributes. `--bucket-classes` names the classes whose capacity backs buckets; `--bucket-endpoint` is
the public S3 URL tenants get (`https` only, unless `--bucket-allow-insecure-endpoint` for local development);
`--bucket-region` (default `default`) is the region clients sign with; `--bucket-admin-endpoint` is the in-cluster
RGW admin URL. Admin credentials come from `AP_BUCKET_ADMIN_ACCESS_KEY` and `AP_BUCKET_ADMIN_SECRET_KEY`. The public
endpoint must refuse `/admin`, or serve an S3-only RGW.

### 5.7 Gateway, CLI, and Flags

Gateway routes on the lease subrouter (`/lease/{dseq}/{gseq}/{oseq}`; the lease ID is built from the authenticated
caller, so only the owner can address it):

| Route                           | JWT scope     | Result                                                              |
| ------------------------------- | ------------- | ------------------------------------------------------------------- |
| `GET /volume`                   | `status`      | phase, size, `import_from`, attached lease, `retained_until`        |
| `GET /bucket`                   | `bucket`      | endpoint, bucket, region, size, usage, keys (never secrets)         |
| `POST /bucket/keys`             | `bucket-keys` | `{name, access}` -> the key, with its secret, once                  |
| `DELETE /bucket/keys/{keyName}` | `bucket-keys` | 204                                                                 |

The bucket routes answer only on the bucket's own lease. New keys need the bucket `Provisioned` under that lease, so
they are refused (409) while it is importing, in reclamation, closed, or being adopted; revoking is always allowed.
The gateway serializes key changes.

Provider CLI: `provider-services volume status --dseq --gseq --oseq --provider`; `provider-services bucket info`;
`provider-services bucket key create <name> [--access readwrite|read]`; `provider-services bucket key revoke <name>`.

Provider daemon flags:

| Flag                                  | Default         | Meaning                                                          |
| ------------------------------------- | --------------- | ---------------------------------------------------------------- |
| `--volume-classes`                    | none            | storage classes offered as volumes (each with a `-retain` class) |
| `--volume-max-size`                   | 0 (no cap)      | largest volume to bid on, in bytes                               |
| `--volume-max-retention`              | 0               | longest retention honored; orders asking more are declined       |
| `--volume-max-replicas`               | 0               | highest replica count honored; 0 declines replication terms      |
| `--reclamation-window`                | 0               | window offered in bids; volume bids need at least the chain floor |
| `--volume-move-bid-timeout`           | 168h            | how long a bid on a re-listed volume order stays open            |
| `--volume-migration-interval`         | 1m              | period between copy-in rounds for a volume moving here           |
| `--volume-transfer-port`              | 8444            | port other providers dial for transfers (the gateway gRPC port)  |
| `--volume-transfer-endpoints`         | none            | `provider=host:port` overrides for where a provider serves transfers |
| `--volume-replication-driver`         | `stream-export` | transfer driver                                                  |
| `--volume-replication-rook-namespace` | `rook-ceph`     | namespace of the rook-ceph-tools pod that runs `rbd`             |
| `--volume-replication-dir`            | OS temp dir     | staging for file-tree copies                                     |
| `--volume-replica-sync-interval`      | 15m             | period between replica pulls (not used yet, §9)                  |
| `--bucket-backend`                    | none            | `rgw` enables buckets                                            |
| `--bucket-classes`                    | none            | classes whose capacity backs buckets                             |
| `--bucket-endpoint`                   | none            | public S3 URL (`https`)                                          |
| `--bucket-region`                     | `default`       | region clients sign with                                         |
| `--bucket-admin-endpoint`             | none            | in-cluster RGW admin URL                                         |

---

## 6. SDL and CLI (`chain-sdk`)

- SDL v2.2 lives in package `sdl` (`go/sdl/v2_2.go`, `groupBuilder_v2_2.go`). `buildVolumeGroup` compiles a
  `volumes:` entry to a single-group deployment: the group named after the placement; one resource unit with
  present-but-zero compute and one Storage entry named after the volume key (the `vid`) with attributes `class` and
  `persistent=true`; and `GroupSpec.Volume = VolumePolicy{vid, reclaim (default retain), retention (default 0),
  max_attachments (1, or 0 for a bucket), max_replicas, adopt, replica_of, access}`. It injects
  `capabilities/storage/volumes: "true"` (or `capabilities/storage/buckets: "true"`) into the placement requirements.
  No lifecycle or `vid` data ever enters `Storage.Attributes`.
- A service-level `volume:` param compiles to `ResourceUnit.Volumes` on chain and `StorageParams.volume` in the
  manifest -- identical content on both sides for the manifest check. A reference is `{external.owner or the signer,
  external.dseq, gseq 1, name = the volume key}`; `external.dseq` is required.
- Grammar enforcement: `volume:` excludes `size`/`class`/`attributes`; mount required; declared-but-unreferenced
  volumes error; `count > 1` with an RWO reference rejected; one volume cannot be referenced by two services; services
  sharing a profile need identical references; `adopt` and `replica-of` are mutually exclusive and need `dseq > 0`; a
  bucket takes no `max-replicas` or `replica-of`; an `external` declaration carries no declaration fields (`size`,
  `class`, `lifecycle`, `max-replicas`, `adopt`, `replica-of`, `access`). A file that both declares and references
  volumes is rejected.
- Fixtures: `testdata/sdl/input/v2.2/{volume-deployment,volume-attach,bucket-deployment}`.
- TS: the schema-generated validator accepts 2.2; the TS manifest generator does not compile volumes yet (§9).
- Tenant CLI (`chain-sdk/go/cli`; node `GetTxCmd` panics by design): `akash tx volume create [sdl-file]`
  (`--dseq --owner --deposit --deposit-sources`) broadcasts the `MsgCreateDeployment` and warns on `retention: 0` and
  on a disk with `max-replicas: 0`; `akash tx volume close` sends `MsgCloseDeployment`; `akash query volume list`
  (`--owner --state --dseq`) and `akash query volume status` (`--owner --dseq`) show the deployment, its groups, the
  escrow account, and the leases. A move is `akash tx market lease close` on the volume lease (the chain applies
  `volume_migrate`). `akash tx market bid create` copies the order's volume policy into the bid; `akash tx deployment
  create|update` pass the signer as the default reference owner.
- Provider client (`go/provider/client`): `LeaseVolume` (`GET .../volume`), `BucketInfo`, `CreateBucketKey`,
  `RevokeBucketKey`. JWT scopes `bucket` and `bucket-keys` (`go/util/jwt`, `specs/jwt-schema.json`, TS
  `validateJwtPayload.ts`).

---

## 7. Upgrade Handler

`node/upgrades/software/v2.2.0/` in the reference implementation (registered in its `init.go`, blank-imported in
`upgrades.go`); the release name is set when the upgrade is scheduled. The same upgrade carries lease resizing behind
its own flag.

- No store migrations: the new fields are additive on the active versions, every module keeps its
  `ConsensusVersion`, and `StoreLoader` adds nothing.
- Seed params where unset: `VolumeOrdersEnabled=false`, `MinVolumeReclamationWindow=24h`, `MaxVolumeSize=32Ti`,
  `MaxVolumeRetention=720h`, `MaxVolumeReplicas=4`.
- Rebuild the `vid`, replica-count, and attachment indexes from state (empty on a network without volumes), as
  `InitGenesis` does on import.
- The escrow-hooks fix (1.3) rides the binary. CHANGELOG entries per convention.

---

## 8. Test Plan

### 8.1 Chain (`node/tests/e2e`, handler and keeper tests)

`tests/e2e/storagemarket_test.go` (no build tag) with the `x/deployment` and `x/market` handler and keeper tests:
flag gating; `vid` uniqueness and adoption gates; colocation and the `CreateLease` re-check; attachment-index
maintenance across every close path; cascade-detach on reclaim completion and on escrow exhaustion; the reclamation
floor in both close directions; close-reason substitution; runway-event math; the hooks fix and the `keeper.go:347`
masking regression (bid-account close byte-identical); grant-funded volume deployments (`AuthorizeDeposits`); the
bucket rules; genesis rebuild of the indexes. Storage-only shape validation (present-but-zero, single group, single
storage entry) is tested in chain-sdk.

### 8.2 Provider e2e (`provider/integration/storagemarket_test.go`)

Registered in `TestIntegrationTestSuite`, modeled on `E2EPersistentStorageDefault`. Kustomize adds a Retain-policy
`rancher.io/local-path` class (`beta3-retain`) to `_docs/kustomize/storage/` plus the node-label append; Volume CRD +
storage operator added to **both** `KUSTOMIZE_INSTALLS` and `RunLocalOperator` (divergence risk between the two is a
known trap). Core loop: create volume -> bid -> lease -> attach compute -> write UUID -> close compute deployment ->
volume re-parks, volume lease still active -> second compute deployment, same ref -> read UUID back.

Money path (same harness): tiny-deposit exhaustion -> cascade-detach -> `volume_unfunded` -> Volume `Retained` ->
adoption deployment -> gated bid -> re-bind -> re-attach -> UUID intact.

### 8.3 Two-Provider Move

A second `RunLocalProvider` (distinct ports and keys) against the same kind cluster, which needs an identical-path
kind mount: close the volume lease -> A `Exporting` -> A closes the lease at the deadline -> re-order -> B wins -> B
`Importing` -> B pulls A's final copy -> `Provisioned` -> re-attach at B -> UUID intact. **Honestly labeled:**
local-path proves the coordination and transfer plumbing, not RBD; the RBD path has no CI coverage yet.

### 8.4 Upgrade CI (to build before release)

- The previous pinned provider release run against the upgraded node: no crash-loop, existing leases keep being
  served and withdrawn, and no volume bid of its lands.
- Test cases for the upgrade in `tests/upgrade/test-cases.json` and an upgrade worker.

### 8.5 Golden Files

The canonical empty-manifest hash (Go); SDL v2.2 -> GroupSpec compilation fixtures; TS/Go parity for v2.2 (the TS
parity suite covers v2.0 and v2.1 today).

---

## 9. Implementation Status

The reference implementation runs end to end on a local sandbox of three providers sharing one kind cluster: disks
and buckets created, leased, and moved between providers; disks attached, detached, and re-attached; adoption after a
close; and the provider e2e suites above. Not built yet:

- **Replica sync.** `RunReplica` is never started, so a replica volume comes up empty; promote tooling.
- **Large RBD moves.** RBD exports pass through a buffered exec in the tools pod and are held in memory, so big RBD
  volumes cannot move yet; the RBD path has no CI coverage.
- **The `rbd-mirror` driver.**
- **Lookups.** A query service for `vid` and attachment lookups, and the CLI conveniences on top of it (`migrate`,
  `adopt`, and `promote` commands, `--accept` strategies, `--vid` resolution).
- **TS manifest generation** for volumes and buckets.
- **The upgrade CI items in 8.4.**
- **Inventory reporting:** `inventory.v1.StorageInfo` has no volume fields; the storage operator reports retained bytes
  on its `/state` endpoint instead.

---

## Appendix A: Repo Hygiene (Phase 0, non-consensus)

- The SDL `GE(2.1.0)` version gate is capped with the schema in lockstep (1.6), with a fixture rejecting 3.0.
- The double `E2EPersistentStorageDefault` registration (`provider/integration/e2e_test.go`) is deduplicated.
- Regenerate the AEP index (`node scripts/index.js` in the AEP repo) once this spec has its number.
