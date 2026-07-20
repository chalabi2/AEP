# AEP-87 Implementation Guide

> Companion to [AEP-87: Decoupled Storage Market](./README.md) (the authoritative specification).
>
> This document contains **implementation-specific details only**: protobuf definitions, store layouts, module and
> provider code changes, the upgrade handler, and the test plan. It does not repeat specification-level content.
> For the design, behavior, rules, and rationale, see [README.md](./README.md).

Target repos: `chain-sdk`, `node`, `provider` (go.work monorepo layout).

---

## 1. Codebase-Specific Implementation Notes

### 1.1 Proto Placement: Staged Version Directories, Decided Once

All consensus-shape changes land in the staged `deployment/v1beta5` and `market/v2beta1` proto directories, activated
by removing them from `chain-sdk/buf.yaml` `excludes`. Active wire types are never touched -- above all
`base.resources.v1beta4.Storage`, which is shared with deployed inventory, manifest, and operator surfaces. The attach
reference (`VolumeRef`) therefore lives on `deployment/v1beta5.ResourceUnit`, not on base resources; `base/resources`
stays at v1beta4.

This makes old-binary safety structural: volume orders and volume-bearing compute orders are `market/v2beta1` types
that deployed bid engines **cannot decode** -- not "can decode but should filter". The rollout does not depend on
unverified `shouldBid` code-path ordering in shipped binaries.

**Phase 0 prerequisite commit (blocking).** The staged dirs diverge from `main`: they lack the reclamation fields the
fork already ships (`Reclamation` on deployment msgs, `Order.Reclamation`, `Bid.ReclamationWindow`) and carry an
unfinished repeated-prices/deposits redesign. Before anything else:

1. Port the reclamation fields into `deployment/v1beta5/deploymentmsg.proto` and
   `market/v2beta1/order.proto`/`bid.proto`.
2. Revert the repeated-prices redesign back to `main`'s active shapes, so activation is a pure superset of
   v1beta4/v2beta5 and the store migrations are re-serialization only.

The staged dirs become "active shapes + AEP-87 additions", nothing more.

**Registration checklist per touched version** (the standard chain-sdk drill): hand-written `codec.go`
(`RegisterInterfaces`; **zero new Msg types**, so no new `RegisterCustomSignerField` entries), `key.go`, `msgs.go`,
`errors.go`, `params.go`; new Query surfaces added to `docs/config.yaml`; `make lint-proto`, `proto-check-breaking`,
`mocks`.

### 1.2 Zero New Msg Types; AuthorizeDeposits Untouched

The complete flow rides existing messages (see the message table in the README). Consequences:

- No `AuthorizeDeposits` switch entries (`MsgCreateDeployment`/`MsgDepositDeployment` are already in it,
  `node/x/escrow/keeper/keeper.go:252-282`).
- **Test obligation (U1):** explicit tests that grant-funded volume deployments work (the `AuthorizeDeposits` switch
  behaves for volume groups).

### 1.3 Escrow Hooks Hardening (prerequisite-grade, ships in U1)

Fix `node/x/market/hooks/hooks.go:31-34`: `OnEscrowAccountClosed` must `return nil` for any XID that fails
`DeploymentIDFromEscrowID`, mirroring the payment hook -- hook errors abort `saveAccount` **mid-write** (refunds and
`store.Set` already applied). AEP-87 adds no new scope and never triggers the bug, but it corrupts state for any future
scope.

**Test obligation:** the foreign-scope hook error is currently masked by `_ = k.ekeeper.AccountClose(...)` at
`node/x/market/keeper/keeper.go:310` swallowing it for `ScopeBid`. Tests must cover the current masking and prove
bid-escrow account-close behavior is byte-identical after the fix -- proven, not assumed.

### 1.4 Canonical Empty Manifest

A storage-only deployment has no services. The canonical volume manifest is a single group with the on-chain group
name and an empty service list; its `Manifest.Version()` (sha256 over SortJSON) is deterministic and version-pinned;
the CLI computes it for `MsgCreateDeployment.Hash`. Manifest v2beta4 `Manifest.Validate()` accepts zero services iff
every paired on-chain group is a volume group. **Golden-file tests across client versions** -- this hash is
consensus-adjacent and gets manifest-hash rigor.

### 1.5 Feature Flag

`market/v2beta1.Params.volume_orders_enabled` (default `false`, governance-flipped) gates volume-group
`CreateDeployment` (checked in `x/deployment` via the existing `dimports.MarketKeeper` interface,
`node/x/deployment/imports/keepers.go:21`) and, defense-in-depth, `CreateBid` on volume orders. Phase 1 ships every
new path unreachable behind it.

### 1.6 SDL Version Gate

`sdl.UnmarshalYAML` (`chain-sdk/go/sdl/sdl.go`) routes `>= 2.1.0` open-endedly to v2.1 today. Change to:
`== 2.0.0 -> v2`, `>= 2.1.0 && < 2.2.0 -> v2_1`, `>= 2.2.0 && < 3.0.0 -> v2_2`; update `sdl-input.schema.yaml` in
lockstep. Add `Volumes()` to `interface SDL`.

---

## 2. Protobuf Definitions

### 2.1 `akash/deployment/v1/volume.proto` (new file, stable package)

New *file* in a stable package is additive-safe; no existing message is reshaped. `VolumePolicy` lives in
`deployment/v1` because it is identity/state vocabulary imported by v1beta5.

```proto
// VolumeRef addresses one Storage entry of a storage-only ("volume") group.
message VolumeRef {
  option (gogoproto.equal) = true;
  string owner = 1 [(cosmos_proto.scalar) = "cosmos.AddressString"];
  uint64 dseq  = 2 [(gogoproto.customname) = "DSeq"];
  uint32 gseq  = 3 [(gogoproto.customname) = "GSeq"];   // always 1 in v1
  string name  = 4;                                      // Storage.Name in the volume group
}

// VolumePolicy marks a group as a storage-only volume group and carries its
// lifecycle contract. Presence of this message IS the volume discriminator.
message VolumePolicy {
  option (gogoproto.equal) = true;

  enum ReclaimPolicy {
    option (gogoproto.goproto_enum_prefix) = false;
    invalid = 0 [(gogoproto.enumvalue_customname) = "VolumeReclaimInvalid"];
    retain  = 1 [(gogoproto.enumvalue_customname) = "VolumeReclaimRetain"];
    delete  = 2 [(gogoproto.enumvalue_customname) = "VolumeReclaimDelete"];
  }

  // vid: tenant-chosen data-continuity label (DNS-label grammar,
  // [a-z0-9]([-a-z0-9]{0,61}[a-z0-9])?). Typed field -- NEVER a storage
  // attribute (attributes flow into AttributesSubsetOf provider matching at
  // x/market/handler/server.go:95 and would make the order unbiddable).
  string vid = 1;

  ReclaimPolicy reclaim = 2;

  // retention: post-close window the provider must keep the data adoptable.
  // Bounded by deployment Params.MaxVolumeRetention; priced into the bid.
  // GC deadline is chain-computable: closedLease.ClosedAt + retention.
  google.protobuf.Duration retention = 3
      [(gogoproto.nullable) = false, (gogoproto.stdduration) = true];

  uint32 max_attachments = 4;   // validation forces 1 in v1 (RWO); RWX is additive
  uint32 max_replicas = 5;      // export duty declared -> priced into the bid

  VolumeRef adopt = 6 [(gogoproto.nullable) = true];       // adoption of a dead retained volume
  VolumeRef replica_of = 7 [(gogoproto.nullable) = true];  // this volume is a replica
}
```

### 2.2 `deployment/v1beta5` (staged dir, activated)

After Phase 0 reconciliation, v1beta5 shapes equal main's v1beta4 plus:

```proto
message GroupSpec {
  // name = 1, requirements = 2, resources = 3 unchanged from v1beta4
  akash.deployment.v1.VolumePolicy volume = 4 [(gogoproto.nullable) = true];
  // nil for every pre-AEP-87 group => migration is re-serialization only
}

message ResourceUnit {
  // resource = 1 (base.resources.v1beta4.Resources -- UNCHANGED base type),
  // count = 2, price = 3 unchanged
  repeated akash.deployment.v1.VolumeRef volumes = 4 [(gogoproto.nullable) = false];
  // externally-leased volumes this unit's services mount; empty pre-AEP-87
}

message Params {
  // existing fields unchanged
  uint64 max_volume_size = N;                       // bytes; volume groups ONLY
  google.protobuf.Duration max_volume_retention = N+1
      [(gogoproto.nullable) = false, (gogoproto.stdduration) = true];
  uint32 max_volume_replicas = N+2;
}
```

Because `market.v2beta1.Order` embeds this GroupSpec, volume policy and attach refs flow into orders -- and therefore
into bidengine `Request`s -- with zero additional market proto surface for the order path.

### 2.3 `market/v1` (stable) -- additive vocabulary only

Identity types (`OrderID/BidID/LeaseID`, `Lease`, `Reclamation`) untouched -- the `mv1.LeaseID(bid.ID)` raw struct
cast and the exactly-5-part payment-XID parse stay safe.

Close reasons (`market/v1/types.proto`, established ranges: 1-9999 owner, 10000+ provider, 20000+ network):

```proto
reason_volume_migrate       = 101;     // owner closes volume lease to migrate
reason_volume_evict         = 10101;   // provider decommissions (via reclaim)
reason_volume_closed_retain = 20101;   // tenant closed volume; retain window runs
reason_volume_unfunded      = 20102;   // escrow exhaustion cascade
reason_volume_detach        = 20103;   // compute lease force-closed by cascade-detach
```

Typed events (`market/v1/event.proto`, emitted via `EmitTypedEvent`):

```proto
message EventVolumeAttached {
  akash.market.v1.LeaseID lease_id = 1 [(gogoproto.nullable) = false]; // compute lease
  akash.deployment.v1.VolumeRef volume = 2 [(gogoproto.nullable) = false];
}
message EventVolumeDetached {
  akash.market.v1.LeaseID lease_id = 1 [(gogoproto.nullable) = false];
  akash.deployment.v1.VolumeRef volume = 2 [(gogoproto.nullable) = false];
  akash.market.v1.LeaseClosedReason reason = 3;
}
```

Deployment event (`deployment/v1/event.proto`):

```proto
message EventVolumeAdopted {
  akash.deployment.v1.GroupID id = 1 [(gogoproto.nullable) = false];        // new volume
  akash.deployment.v1.VolumeRef adopted = 2 [(gogoproto.nullable) = false]; // dead volume
  string vid = 3;
}
```

### 2.4 `market/v2beta1/params.proto` (staged dir, activated)

```proto
message Params {
  // existing (incl. Min/MaxReclamationWindow) unchanged
  bool volume_orders_enabled = N;      // THE feature flag; default false, gov-flipped
  google.protobuf.Duration min_volume_reclamation_window = N+1
      [(gogoproto.nullable) = false, (gogoproto.stdduration) = true];
}
```

### 2.5 `escrow/v1/event.proto` (the only escrow-package touch; no state change)

```proto
// Emitted at every settle of an account with open payments: the deterministic,
// tenant-computable exhaustion deadline.
message EventAccountRunway {
  akash.escrow.id.v1.Account id = 1
      [(gogoproto.nullable) = false, (gogoproto.customname) = "ID"];
  int64 projected_depletion_height = 2;   // 0 = funds cover no full block
}
```

### 2.6 Manifest `v2beta4` (new version dir under `proto/provider/.../manifest`)

```proto
message StorageParams {
  string name = 1;
  string mount = 2;          // mount topology stays off-chain, as today
  bool read_only = 3;
  // volume mirrors the on-chain VolumeRef as "owner/dseq/gseq/name";
  // empty for locally-provisioned (AEP-15) storage.
  string volume = 4;
}
```

`Service.checkAgainstGSpec` in the v2beta4 path requires the set of manifest `StorageParams.volume` refs to equal the
on-chain `ResourceUnit.Volumes` exactly (index-wise, after canonical sort by name). Manifest v2beta3 behavior is
byte-identical -- every new field exists only under new version numbers, preserving determinism at both load-bearing
points (manifest sha256/SortJSON hash; GroupSpec construction order).

### 2.7 `VolumeTransfer` gRPC (`proto/provider/akash/volume/v1`; data plane, off-chain)

grpc-gateway pattern per `inventory/v1`; served by the storage operator on the provider gateway:

```proto
service VolumeTransfer {
  rpc Export(ExportRequest) returns (stream ExportChunk);   // full or diff
  rpc Status(StatusRequest) returns (StatusResponse);       // sync lag, snapshots held
}
message ExportRequest {
  string owner = 1;
  string vid = 2;
  uint64 dseq = 3;               // the volume deployment on the EXPORTER's side
  string requester = 4;          // destination provider address (must match mTLS cert)
  string from_snapshot = 5;      // "" = full; else diff base (resumable/incremental)
  uint64 resume_offset = 6;
}
message ExportChunk { bytes data = 1; bytes sha256 = 2; uint64 offset = 3; }
```

mTLS with x/cert on-chain provider identities, both directions; authorization derived from chain state on both ends
(see README [Migration](./README.md#migration)); per-chunk sha256 + whole-image digest in `StatusResponse`.

---

## 3. Store Layout (no new store keys; indexes inside existing modules)

```
x/deployment store (existing key):
  0x11  deployments   IndexedMap (existing)
  0x12  groups        IndexedMap (existing) -- the volume's durable spec-of-record
  0x13  volumesByVid  Map[Pair[owner, vid] -> GroupID]                     (NEW)
        maintained on volume create / close / adopt; enforces per-owner vid
        uniqueness among live volumes; resolves vid -> current volume

x/market store (existing key):
  0x12 orders, 0x13 bids, 0x14 leases (existing)
  0x15  attachments     KeySet[Pair[GroupIDKey(volume), LeaseIDKey(compute)]]  (NEW)
  0x16  attachmentsRev  KeySet[Pair[LeaseIDKey(compute), GroupIDKey(volume)]]  (NEW)
        written/pruned ONLY inside the keeper (never handlers) so every close
        path -- tenant, provider, escrow cascade -- maintains them; iteration
        is deterministic (key order)
```

Both indexes ship with genesis export/import and ride the module `ConsensusVersion` bump migrations (empty at upgrade)
from day one -- the network-upgrade CI job blocks release otherwise. `StoreLoader.Added` stays empty: no
store-key-mount-on-exact-upgrade-name hazard. A small keeper counter beside the vid index tracks `replica_of` backref
counts per volume.

---

## 4. Chain Module Changes (`node`)

No new module, no new stores, no new hooks, no EndBlocker anywhere. `app/types/app.go`, `app_configure.go`
(`orderInitGenesis`), `modules.go`, `mac.go`, `SetupHooks` -- all untouched.

### 4.1 chain-sdk v1beta5 Go validation (ships in U1, inert behind the flag)

`validateResources` / `ResourceUnits.Validate` (`chain-sdk/go/node/deployment/v1beta5/resourceunit.go`):

- `GroupSpec.Volume != nil` (volume groups): CPU/Memory/GPU **present-but-zero** (fields non-nil, values zero -- every
  downstream dereference stays crash-safe); exactly one Storage entry with `persistent=true`, `class != ram`, size in
  `[5Mi, params.MaxVolumeSize]`, `count == 1`; `ResourceUnit.Volumes` empty (a volume never references one); `vid`
  grammar; `retention <= MaxVolumeRetention`; `max_attachments == 1`; `max_replicas <= MaxVolumeReplicas`;
  `adopt`/`replica_of` mutually exclusive and stateless-validated.
- `GroupSpec.Volume == nil` (compute groups): existing rules unchanged, plus each `VolumeRef` stateless-validated;
  **no Storage entry corresponds to a VolumeRef** -- attached volumes contribute zero storage quantity, never
  double-counted in compute inventory or pricing.
- `ValidateDeploymentGroups`: a deployment containing a volume group contains **only** that group (single-group
  invariant => `gseq` always 1; escrow account == volume).

### 4.2 `x/deployment` (`node/x/deployment/handler/server.go`, `keeper/`)

`CreateDeployment`, after existing validation, when the single group is a volume group:

1. Feature flag: reject unless `market.Params.VolumeOrdersEnabled`.
2. **vid gate:** look up `volumesByVid[owner, vid]` -- live volume => `ErrVidInUse`; retained dead volume within its
   retention window => `ErrVidRetained` unless `VolumePolicy.adopt` names exactly that volume; expired/absent =>
   fresh create, index (over)written.
3. Adoption validation (`adopt` set): referenced group exists, is a volume group, owner matches, is closed, vid
   matches, block time within the dead volume's retention window (read from its last closed lease's `ClosedAt` --
   additive `closed_at` on the v1 lease record if not already present). Emits `EventVolumeAdopted`.
4. Replica validation (`replica_of` set): target exists, is a live volume group, owner matches, backref count <
   its `max_replicas`.
5. Proceed exactly as today: `market.CreateOrder(...)` then
   `escrow.AccountCreate(deployment.ID.ToEscrowAccountID(), ...)` -- the volume's migration window rides the
   existing `Reclamation` field of `MsgCreateDeployment`.

`CloseDeployment`/`CloseGroup` on a volume: reject with `ErrVolumeAttached` while the attachment index has entries
(narrow market-keeper method `HasAttachments(groupID)`); tenants close compute first. The escrow cascade cannot be
blocked -- it cascade-detaches instead (4.4).

### 4.3 `x/market` (`node/x/market/handler/server.go`, `keeper/`)

`msgServer.CreateBid` -- additive checks after the existing reclamation block (`handler/server.go:104-121`):

1. Volume-group bids: flag check; `msg.ReclamationWindow >= params.MinVolumeReclamationWindow`; `ResourcesOffer`
   storage quantities **equal** the order's per-name quantities (closes the `MatchGSpec` boundary TODO,
   `chain-sdk/go/node/market/v2beta1/bid.go`, *for volume orders only* -- the global compute fix is out of scope);
   capability matching rides `order.MatchResourcesRequirements` unchanged (the SDL injects
   `capabilities/storage/volumes: "true"` into `PlacementRequirements`). Adoption orders: additionally
   `msg.ID.Provider == provider of the dead volume's most recent closed lease` -- pure chain state.
2. Attaching-group bids (any `ResourceUnit.Volumes` non-empty), per ref: group exists, is a volume group,
   `ref.owner == order.ID.Owner`; an **active** lease exists on the volume group with
   `lease.ID.Provider == msg.ID.Provider` (colocation -- reads *current* state, so it survives the auto-re-order loop
   and migration for free); attachment count < `max_attachments` -- O(1) prefix count, not a lease scan.

`msgServer.CreateLease` -- attaching groups: **re-verify** colocation and the attachment bound, then write both index
directions and emit `EventVolumeAttached`. Volume groups: nothing special.

**Shared keeper close routine** (reached by `MsgCloseLease`, `MsgCloseBid`, and escrow hooks via `OnGroupClosed`), at
the single point where `EventLeaseClosed` is emitted:

- Closing lease in `attachmentsRev` (a compute attach): prune both directions, emit `EventVolumeDetached{reason}`.
- Closing lease is a volume lease: cascade-detach then reason substitution (4.4).

**Tenant volume close routes through reclamation:** `MsgCloseDeployment` on an unattached volume and `MsgCloseLease`
on a volume lease enter `LeaseReclaiming` with
`deadline = now + max(bid.ReclamationWindow, params.MinVolumeReclamationWindow)`; the close completes at msg time after
the deadline (the existing `MsgCloseBid` state gate, generalized to the tenant direction for volume groups).

**Auto-re-order carve-out:** the existing still-open-group re-order loop is kept for `volume_migrate`/`volume_evict`
(it *is* the migration re-listing); terminal closes (`volume_unfunded` => group `insufficient_funds`; deployment
closed) fire no re-order -- adoption takes over.

### 4.4 Cascade-Detach (keeper close routine, every path)

Invariant: a volume lease never reaches `closed` while the attachment index holds entries for its group.

1. Reclaim completion (provider `MsgCloseBid` after deadline; tenant close after its window): iterate the volume's
   attachments in deterministic key order, force-close each attached compute lease with `reason_volume_detach` (normal
   compute close path per lease: payment closed, event emitted, its group auto-re-orders), then close the volume lease
   with the substituted volume reason.
2. Escrow exhaustion (`OnEscrowAccountClosed -> close deployment -> OnGroupClosed`): same cascade-detach inline, then
   close with `reason_volume_unfunded`. Deterministic, single-tx.
3. Final assert-empty on the volume's attachment entries guards the invariant.

Bounded: at most `max_attachments` (=1 in v1) extra closes per volume -- O(1), gas-accounted.

### 4.5 Escrow (`node/x/escrow/keeper/keeper.go`)

- `EventAccountRunway` emitted at the end of `accountSettle` for any account with open payments:
  `projected_depletion_height = current + floor(Funds[0] / sum(open payment rates))`. No state change; generic across
  scopes.
- Preserved by not touching: the d7d0205d payments-before-account save ordering; `AccountDeposit`/overdraw semantics;
  AEP-75 FIFO drain and grant re-credit; single-denom uact settlement.

---

## 5. Provider Changes

### 5.1 StorageClasses

The storage-operator Helm chart installs akash-labeled `-retain` StorageClasses per class (`beta1-retain` ...
`beta3-retain`; `reclaimPolicy: Retain`, same provisioners -- the `akash-nodes-storageclass.yaml` precedent) and
appends them to the `akash.network/storageclasses` node label. **Invariant: every PV backing a volume group is created
`reclaimPolicy: Retain` at provision time.** The AEP-15 per-lease path keeps its Delete classes untouched.

### 5.2 Storage Operator (`provider/operator/storage/`)

New cobra subcommand (one `AddCommand` line in `operator/cmd.go`; clientsets, errgroup, pubsub free from the parent
PreRun) -- the CRD-reconciler archetype (hostname/ip operator model). Replicas:1; all state re-derivable from
CRDs + PVs + chain queries; nothing only-in-memory.

**Volume CRD** (`pkg/apis/akash.network/v2beta2`, modeled on `provider_host.go`; clientset regenerated; added to
`crd.yaml` upload paths). Name: `volume-<sha256(owner/vid)[:12]>`. Spec: owner, vid, groupId, class, size, reclaim,
retention, leaseId. Status: `phase` (`Pending|Provisioned|Attached|Retained|Adopting|Exporting|Releasing`), `pvName`,
`attachedLease`, `retainedUntil` (= closedAt + retention -- the GC ledger).

**PV choreography -- never through `Available`** (PVCs are namespaced; PVs are cluster-scoped; the volume's durable
kube object is the PV):

- *Provision*: holder PVC `<crd-name>` in namespace `akash-volumes` against the `-retain` class, pinning
  `spec.volumeName` once bound; PV labeled `akash.network/volume-owner|vid|dseq`. State Provisioned/Parked.
- *Attach*: create the target PVC in the compute lease namespace **first** (with `spec.volumeName`), patch
  `pv.spec.claimRef` **directly to the target PVC** (name+namespace+UID), then delete the holder PVC. The PV is never
  an unclaimed `Available` Retain PV the kube binder could hand to an arbitrary pending PVC -- the cross-tenant
  re-bind race is closed by construction.
- *Detach* (compute namespace deleted; PV `Released`): recreate holder PVC, patch `claimRef` back. Parked.
- *Retain*: stamp `retainedUntil`. *Adopt*: verify chain state (the adoption deployment's `VolumePolicy.adopt` matches
  this CRD's dead groupId), update `spec.groupId/leaseId`, re-park. A Released PV is never adopted for a different
  owner.
- *GC*: delete PV + RBD image only when `now > retainedUntil` **and** phase not in {Exporting, Adopting};
  `reclaim: delete` volumes GC immediately after close+detach. **The single destruction path**, driven by a positive
  chain-computable deadline -- never by `EventLeaseClosed` inference.

**Chain watch:** subscribes to `EventLeaseClosed` (volume reasons), `EventVolumeAttached/Detached`,
`EventVolumeAdopted`; re-lists on restart with its **own** pagination checkpoint key -- never sharing the bidengine's
`GetOrdersNextKey`.

**Daemon wiring:** storage-operator client under `cluster/kube/operators/clients/`, `clfromctx` key, appended to
`waitClients` as `waiter.Waitable` -- mandatory only when the provider advertises `capabilities/storage/volumes`
(config-gated), so non-participants gain no chart-before-daemon ordering constraint. Helm chart cloned from
`akash-inventory-operator` with matching discovery labels; RBAC adds PV/PVC create/delete, `volumes.akash.network`,
`snapshot.storage.k8s.io`, and rook-ceph-tools exec. Rook absence tolerated via the `crdInstalled` polling pattern.

**Inventory truth (three legs, all required):** (1) volume allocations published as long-lived allocated entries into
the inventory operator's merged cluster state (a `storageSignal` on the existing merge loop) so parked/retained volumes
are never double-counted as free; (2) rebuilt from Volume CRDs on restart; (3) `inventory.v1.StorageInfo` extended
additively only (`volumes_capable bool`, `retained_bytes uint64`).

### 5.3 Cluster Layer (`provider/cluster`, `cluster/kube`)

- `cluster.Client` (`cluster/client.go:63`) gains `DeployVolume / AttachVolume / DetachVolume / TeardownVolume /
  VolumeStatus` (nullClient stubs included); implementations delegate to the Volume CRD -- the daemon never
  manipulates PVs directly.
- `Deploy`'s persistent switch (`kube/client.go:~570`) gains a third case: a service with a volume ref renders as a
  k8s `Deployment` (not StatefulSet -- positional per-replica VolumeClaimTemplates don't apply), `replicas=1`, pod
  volume by pre-bound `ClaimName` -- resolving the `DataSource: nil` seam in `Workload.persistentVolumeClaims()`
  (`kube/builder/workload.go:226`).
- `TeardownLease` is **unchanged** -- namespace deletion removes only the attach PVC; the Retain PV survives and the
  operator re-parks it. Deploy-failure rollback is equally harmless.
- Volume objects carry `akash.network/component: volume` labels excluded from `cleanupStaleResources`' selector.
- Reservations: `ctypes.Reservation` gains a **durable** kind -- counted by `inventory.Adjust`'s per-class `SubNLZ`,
  `StorageCommitLevel` forced to 1.0 (durable leased bytes never thin-sold), exempt from unreserve-on-lease-close
  (released only on volume GC), rebuilt on restart from Volume CRDs. `inventoryService.reserve`'s nil-unit
  hard-rejects (`cluster/inventory.go:185`) are satisfied by the present-but-zero shape -- no relaxation needed.

### 5.4 Bid Engine (`provider/bidengine`)

- **Discovery: zero changes.** Volume orders are ordinary `EventOrderCreated`/`OrderOpen` (v2beta1); `newOrder`
  (`order.go`) forks on `gspec.Volume != nil`.
- **Volume `shouldBid`:** skip CPU/GPU/endpoint gates; keep placement/audit gates; require the class in advertised
  capabilities **and** live per-class headroom from the retained `inventory.Cluster` (closing the
  `provider_attributes.go:162` TODO for the storage path); require the `-retain` class installed; check provider caps
  (max size, max retention honored, replication support). Adoption orders: local pre-check for a Retained Volume CRD
  matching owner/vid (the chain gate is the authority).
- **Pricing:** `scalePricing` per-class map applies (compute terms zero); `shellScriptPricing`'s `storageElement`
  gains additive `volume, retention_hours, max_replicas, replica` fields (+ `run.go` flags).
- **Win handoff:** `VolumeLeaseWon` bus event -> volume provisioner (creates the Volume CRD; **no manifest wait** --
  the GroupSpec is the whole contract). `checkForExistingBid` restart recovery applies unchanged.
- **Attach orders:** ordinary compute path plus a local check that each ref matches a Parked Volume CRD; attach
  storage contributes zero quantity -- no reservation double-count.

### 5.5 Replication Drivers

`ReplicationDriver` interface; two implementations:

1. **`stream-export` (default, required):** CSI `VolumeSnapshot` -> `rbd export` / `rbd export-diff` (via the existing
   `RemotePodCommandExecutor` against rook-ceph-tools; plain file copy for local-path) streamed over `VolumeTransfer`.
   No Ceph cluster peering, no WAN-exposed mons, no pool-scoped cephx credentials. The mainnet posture.
2. **`rbd-mirror` (opt-in):** snapshot-mode rbd-mirror for mutually trusting provider pairs, only with per-volume RBD
   namespaces (or dedicated pools) and namespace-scoped peer users -- bootstrap peer tokens are pool-level credentials
   and treated as such. Disabled by default.

Migration flow: on `EventLeaseReclaimStarted{volume_evict|volume_migrate}`, source CRD -> Exporting (GC frozen);
destination pulls full export, then periodic diffs, final `export-diff` after cascade-detach/quiesce; destination flips
its CRD to Parked. If import fails, the destination closes its bid and the order re-lists; the source holds data
through window + retention.

---

## 6. SDL and CLI (`chain-sdk`)

- SDL v2.2 package (`sdl/v2_2`): top-level `volumes:` stanza; `buildGroups()` compiles a volume entry to
  `GroupSpec{Volume: &VolumePolicy{...}, Resources: [present-but-zero + one Storage, count 1]}` and injects
  `capabilities/storage/volumes: "true"` into `PlacementRequirements`. No lifecycle/vid data ever enters
  `Storage.Attributes`. Service-level `volume:` params compile to `ResourceUnit.Volumes` refs on-chain and
  `StorageParams.volume` strings in the v2beta4 manifest -- identical content both sides for `checkAgainstGSpec`.
- Grammar enforcement: `volume:` excludes `size`/`class`/`attributes`; mount required; declared-but-unreferenced
  volumes error; `count > 1` with an RWO ref rejected.
- CLI (`chain-sdk/go/cli`; node `GetTxCmd` panics by design): `akash tx volume create -f volume.sdl
  [--accept lowest-audited|manual] [--deposit ...]` (broadcast, stream bids, auto-accept per strategy; `--no-accept`
  for manual); `akash tx volume close|migrate|adopt`; `akash query volume list|status --vid` (resolves the 0x13
  index; shows lease state, attachment, runway from `EventAccountRunway`, retention deadline);
  `akash volume promote --replica dseq` (close attach -> redeploy against the replica). Warn when `max-replicas: 0`.

---

## 7. Upgrade Handler

`node/upgrades/software/vX.Y.0/` (blank-imported in `upgrades.go`):

- `RegisterMigration(deployment, 7 -> 8)` and `RegisterMigration(market, 8 -> 9)`: re-encode stores to
  v1beta5/v2beta1 (nullable-additive => re-serialization only).
- Seed params: `VolumeOrdersEnabled=false`, `MinVolumeReclamationWindow=24h`, `MaxVolumeSize`,
  `MaxVolumeRetention=720h`, `MaxVolumeReplicas=4`.
- Initialize the empty 0x13/0x15/0x16 indexes; genesis export/import included in the same release.
- The escrow-hooks fix (1.3) rides the binary.
- `StoreLoader{Added: nil}` -- no new store keys. ConsensusVersion bumps logged in CHANGELOG per convention.

**Old-binary verification job (must pass, not asserted):** the network-upgrade CI job additionally runs the previous
pinned provider release against the upgraded node and asserts no crash-loop, existing leases keep being served and
withdrawn, and no bids are placed on v2beta1 orders.

---

## 8. Test Plan

### 8.1 Fast, k8s-free (`node/tests/e2e`)

Flag gating; storage-only validation shapes (present-but-zero, single group, single storage entry); vid
uniqueness/adoption gates; colocation + CreateLease re-verify; attachment-index maintenance across every close path;
cascade-detach on reclaim completion and on escrow exhaustion; reclamation-floor enforcement in both close directions;
close-reason substitution; runway-event math; hooks fix + `keeper.go:310` masking regression (bid-account close
byte-identical); grant-funded volume deployments (`AuthorizeDeposits`).

### 8.2 Provider e2e (`provider/integration/storagemarket_test.go`)

Registered in `TestIntegrationTestSuite`, modeled on `E2EPersistentStorageDefault`. Kustomize adds a Retain-policy
`rancher.io/local-path` class (`beta3-retain`) to `_docs/kustomize/storage/` plus the node-label append; Volume CRD +
storage operator added to **both** `KUSTOMIZE_INSTALLS` and `RunLocalOperator` (divergence risk between the two is a
known trap). Core loop: create volume -> bid -> lease -> attach compute -> write UUID -> close compute deployment ->
PV re-parks, volume lease still active -> second compute deployment, same ref -> read UUID back.

Money paths (same harness): tiny-deposit exhaustion -> cascade-detach -> `volume_unfunded` -> CRD Retained ->
adoption deployment -> gated bid -> re-bind -> re-attach -> UUID intact. Runway event asserted at each settle.

### 8.3 Two-Provider Migration

Second `RunLocalProvider` (distinct ports/keys) against the same kind cluster: reclaim -> export (full-copy over
`VolumeTransfer`, local-path driver) -> auto-re-order -> import -> re-attach at provider B -> UUID intact. **Honestly
labeled:** local-path proves the coordination and transfer plumbing, not RBD; `rbd export-diff`/rbd-mirror is validated
on a real two-cluster Rook pair via a manual `_run/` runbook, out-of-band, not hidden behind green CI.

### 8.4 Upgrade CI

The semver upgrade directory exercises the U1 migrations and the old-provider-binary verification job automatically.
Genesis export/import round-trips for the new indexes block release if missing.

### 8.5 Golden Files

Canonical empty-manifest hash across client versions; SDL v2.2 -> GroupSpec compilation snapshots; v1beta4 -> v1beta5
store re-serialization fixtures.

---

## Appendix A: Repo Hygiene (Phase 0, non-consensus)

- Dedupe the double `E2EPersistentStorageDefault` registration (`provider/integration/e2e_test.go:752-753`).
- Cap the SDL `GE(2.1.0)` version gate + schema lockstep (1.6) before shipping the v2.2 parser.
- Regenerate the AEP index (`node scripts/index.js` in the AEP repo) after landing this spec.
