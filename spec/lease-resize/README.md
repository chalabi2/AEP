---
title: "In-Place Lease Resize"
author: Joseph Chalabi (@chalabi2)
status: Draft
type: Standard
category: Core
created: 2026-10-29
---

## Summary

This AEP lets a tenant change the resources of an active lease without closing it. The lease keeps its ID, its
provider, its namespace, and its endpoints; only its resources and its price change. It defines:

1. **Resize by consent.** Four messages: the tenant requests new resources for a lease (`MsgResizeLease`), the
   lease's provider accepts at a new price (`MsgAcceptLeaseResize`) or rejects (`MsgRejectLeaseResize`), and the
   tenant can withdraw the request (`MsgCancelLeaseResize`). Nothing about the lease changes until the provider
   signs the accept, and the provider can never charge more than the tenant's price ceiling.
2. **What can change.** CPU, memory, GPU count, ephemeral storage, replica count, and the price ceiling, up or down.
   Volumes and buckets of the Decoupled Storage Market can grow. Everything that changes scheduling or networking
   stays fixed.
3. **Exact billing.** At the accept, escrow settles the lease at the old rate through that block and charges the new
   rate from the next one. No block is ever re-priced.
4. **Request identity.** The chain stamps each request with a digest, and an answer must name the digest of the
   request it answers, so a quote can never be applied to a different request.
5. **Provider admission and apply.** The provider judges a resize the way it judges a bid: the same gates, a
   capacity check, and its own pricing. Compute leases roll their pods in place once the tenant sends the new
   manifest; volumes and buckets grow in place.
6. **Rollout.** The feature ships dark behind `lease_resize_enabled` and is turned on by governance.

## Motivation

A lease's resources are fixed for its whole life. `MsgUpdateDeployment` only replaces the manifest hash, the CLI
refuses any change to a group, and the provider accepts a manifest only if its resources equal the leased ones
exactly. A tenant who needs more CPU, more memory, or a bigger disk has one option: close the lease and deploy again.
That means a new lease, often a new provider, a new namespace, new endpoints, and the loss of anything stored in the
lease. The reverse is just as bad: a tenant paying for capacity it no longer uses must redeploy to pay less, so many
simply keep overpaying.

Every cloud the network competes with lets a tenant change a machine's size while keeping its disks, its address, and
its identity. Workloads that grow (databases, chain nodes, inference servers whose traffic rises) are exactly the ones
that can least afford a redeploy. With the Decoupled
Storage Market their data can already outlive a lease; this AEP removes the other reason to redeploy, so a tenant can
grow the compute in front of that data, or the volume itself, without moving.

## Specification

### Messages

All four are `akash.market.v1beta5` messages. `resources` uses `akash.deployment.v1beta4.ResourceUnit`, the same type
a deployment's group spec carries.

| Message                | Fields                                                     | Signer         | Effect                                   |
| ---------------------- | ---------------------------------------------------------- | -------------- | ---------------------------------------- |
| `MsgResizeLease`       | `id` (LeaseID), `resources` (the group's new units), `hash` (bytes) | lease owner    | stores a pending resize                  |
| `MsgAcceptLeaseResize` | `id`, `price` (DecCoin), `digest` (bytes)                  | lease provider | applies the resize at `price`            |
| `MsgRejectLeaseResize` | `id`, `reason`, `digest`                                   | lease provider | clears the pending resize                |
| `MsgCancelLeaseResize` | `id`                                                       | lease owner    | clears the pending resize                |

```proto
message MsgResizeLease {
  akash.market.v1.LeaseID id = 1;
  repeated akash.deployment.v1beta4.ResourceUnit resources = 2;  // at least one
  bytes hash = 3;                                                // empty or 32 bytes: the new manifest hash
}
message MsgAcceptLeaseResize {
  akash.market.v1.LeaseID id = 1;
  cosmos.base.v1beta1.DecCoin price = 2;                         // > 0
  bytes digest = 3;                                              // 32 bytes: the request being answered
}
message MsgRejectLeaseResize {
  akash.market.v1.LeaseID id = 1;
  akash.market.v1.LeaseResizeCancelReason reason = 2;            // a provider reason (>= 100)
  bytes digest = 3;
}
message MsgCancelLeaseResize {
  akash.market.v1.LeaseID id = 1;
}
```

### What a Resize May Change

A request carries the group's complete new list of resource units. The chain compares it with the group's stored
spec and merges it; any violation fails with `ErrInvalidResize`.

| Part of the group spec                           | Rule                                                                            |
| ------------------------------------------------ | ------------------------------------------------------------------------------- |
| Resource units                                   | the same units: same number, same IDs, no duplicates (any order; stored order kept) |
| CPU units, memory quantity                       | may go up or down; must stay present; attributes must stay the same set         |
| GPU count                                        | may change on a unit that has GPUs; GPUs cannot be added to or removed from a unit; GPU attributes (vendor, model) must stay the same set |
| Ephemeral and `ram` storage quantity             | may go up or down                                                               |
| Persistent storage on a compute group            | fixed                                                                           |
| Storage entries                                  | same number, same names in the same order, same attributes (so the same class)  |
| `count` (replicas)                               | may change, except on a unit whose services mount a volume, where it is frozen  |
| Per-unit price (the tenant's ceiling)            | the amount may change; the denom is fixed                                       |
| Endpoints, volume references                     | must stay identical                                                             |
| Group name, placement requirements, `VolumePolicy` | carried over from the stored spec; not part of the request                    |
| A storage-market volume or bucket                | its single storage entry may only grow (see [Volumes and Buckets](#volumes-and-buckets)) |

A request that changes nothing fails. A change to the price ceiling alone is a change. The merged spec must pass the
group spec's own validation.

### Request

`MsgResizeLease` succeeds when, in order:

1. `lease_resize_enabled` is on (`ErrLeaseResizeDisabled`);
2. the lease exists and is active (a lease in reclamation is not active);
3. the merged spec passes the rules above;
4. for a volume group: `hash` is empty, the new size is within `max_volume_size`, and the volume is at neither end of
   a live replication;
5. the deployment has no unexpired pending resize, for this lease or any other (`ErrLeaseResizePending`).

The chain stores the request as the deployment's pending resize:

```proto
message LeaseResize {
  akash.market.v1.LeaseID lease_id = 1;
  repeated akash.deployment.v1beta4.ResourceUnit resources = 2;   // the merged units
  bytes hash = 3;
  cosmos.base.v1beta1.DecCoin max_price = 4;                      // sum of price x count over the merged units
  int64 requested_at = 5;                                         // block height
  int64 expires_at = 6;                                           // unix seconds: block time + lease_resize_ttl
  bytes digest = 7;                                               // SHA-256 of this record with digest empty
}
```

and emits `EventLeaseResizeRequested{id, max_price, expires_at, digest}`. `max_price` has the meaning of
`GroupSpec.Price()`: the most the tenant will pay per block for the whole lease at the new size.

**`hash`** is the hash of the manifest the tenant will deploy at the new size. A compute group's manifest carries its
per-service resources, so a real resize changes it; when the request carries the new hash, the accept installs it as
the deployment's hash. A compute request may leave it empty, in which case the tenant updates the deployment's hash
after the accept and before sending the new manifest. A volume group's request must leave it empty: a volume's size is
not part of its canonical empty manifest.

**One pending resize per deployment.** The manifest hash covers every group of a deployment, so two concurrent
resizes in one deployment would each carry a hash that ignores the other's change. The pending resize is therefore
keyed by deployment, holding the lease ID inside. While an unexpired resize is pending, `MsgUpdateDeployment` fails
with `ErrLeaseResizePending`: the accept sets the deployment's hash, and an update landing in between would be
silently overwritten.

**Expiry** is lazy and time-based. A request expires once block time passes `expires_at`. There is no end-block work:
an expired record stays until the next request for the deployment overwrites it, an answer or cancel removes it, or
the lease closes. Storage is bounded at one record per deployment.

### Accept

`MsgAcceptLeaseResize`, signed by the lease's provider, applies the resize in one message, fully or not at all:

1. A pending resize exists for this lease (`ErrLeaseResizeNotFound`) and its digest equals the message's
   (`ErrLeaseResizeChanged`).
2. It has not expired (`ErrLeaseResizeExpired`), and the lease is still active.
3. `price` uses the lease's denom, and `0 < price <= max_price` (`ErrLeaseResizePriceTooHigh`).
4. The merged spec is re-derived against the group as stored now; for a volume group the size cap and the
   live-replication rule are checked again, so a replica created after the request blocks the accept.
5. **Escrow switches rate** ([Escrow](#escrow)): the lease's payment is settled at the old rate through this block,
   the accept fails if the account is overdrawn, and the payment's rate becomes `price`.
6. Both copies of the spec are rewritten: the group's spec (`x/deployment`) and the active order's spec
   (`x/market`), which the market reads for re-orders and for the volume attachment checks.
7. `Lease.Price = price`, and the active bid's price is set to match (the lease ID is the bid ID).
8. If the request carries a hash that differs from the deployment's, the deployment's hash becomes it and
   `EventDeploymentUpdated{id, hash}` is emitted, so the provider accepts the new manifest.
9. The pending resize is removed and `EventLeaseResized{id, price, prev_price, digest}` is emitted.

### Reject, Cancel, and Close

- `MsgRejectLeaseResize` (the provider, naming the request's digest) and `MsgCancelLeaseResize` (the tenant) remove
  the pending resize and emit `EventLeaseResizeCanceled{id, reason, digest}`. A reject works on an expired record too.
- Every lease close, a provider's `MsgLeaseStartReclaim`, and a tenant's close of a storage-market volume lease
  remove the lease's pending resize with reason `lease_closed`.

| Reason (`LeaseResizeCancelReason`) | Value | Set by                                         |
| ---------------------------------- | ----- | ---------------------------------------------- |
| `owner`                            | 1     | the tenant's cancel                            |
| `lease_closed`                     | 2     | a close or reclaim of the lease                |
| `reject_capacity`                  | 100   | provider: no capacity for the new size         |
| `reject_price`                     | 101   | provider: its price is above `max_price`       |
| `reject_unsupported`               | 102   | provider: a change it cannot make              |
| `reject_other`                     | 103   | provider: any other reason                     |

### Request Identity

The chain computes each request's digest, the SHA-256 of the stored record's encoding with the digest field empty,
and emits it with all three resize events. Accept and reject must name it. Without it, a tenant who canceled and
requested again (even in the same block, so with the same expiry) while the provider was evaluating would have the new
request applied at the price the provider worked out for the old one. Providers copy the digest from the chain rather
than recomputing it, so a provider built against an older encoding still names it exactly.

### Escrow

Settlement is lazy: an account is charged `rate x blocks` at its current rate whenever it is settled. Changing the rate
without settling first would re-price every unsettled block. The accept therefore uses a new escrow keeper method,
`PaymentUpdateRate(payment, rate)`:

1. check the payment is open, the denom matches, and the new rate is positive;
2. settle the account at the old rate through this block, and fail if it is overdrawn;
3. set the payment's rate, save the payment and the account, and emit the account's runway event
   (`EventAccountRunway`, defined by the Decoupled Storage Market AEP) at the new rates.

No deposit travels with a resize. A tenant who wants more runway at the new price deposits with `MsgAccountDeposit`.

### Volumes and Buckets

With the Decoupled Storage Market, a volume or bucket is a storage-only group with a single storage entry. A resize can
grow that entry, and nothing else about a volume changes (its compute stays zero and its count stays 1):

- the new size must be larger and within `max_volume_size`; a volume never shrinks;
- the request carries no hash;
- a volume at either end of a live replication (a source with replicas, or a replica whose source is live) cannot be
  resized, so a replica is never smaller than what it copies;
- because a volume can grow, an adoption or a replica must be at least the current size of the volume it takes over
  or copies.

A bucket grows the same way; its provider raises the bucket's quota.

### Provider Responsibilities

**Admission.** A provider learns of a request from `EventLeaseResizeRequested` for one of its leases, and lists its
pending requests when it starts and every minute after, so a request whose event it missed is still answered. It
re-reads the request and stops without answering if the request is gone, its
digest has changed, or it has expired. Otherwise it judges the merged spec the way it judges a bid:

1. the merged spec must apply cleanly, or it rejects `reject_unsupported`;
2. the same gates as a bid (attributes, capabilities including GPU, auditors); for a volume or bucket, its volume
   terms, plus, for a disk, a class the provider can expand in place (a bucket grows by a quota raise, which needs no
   expansion); at startup the provider drops any configured class whose Retain StorageClass does not allow volume
   expansion, so it never accepts a growth it cannot apply;
3. capacity for the full new spec, alongside the lease's current pods (the old pods are not credited back, so this
   never over-commits against the provider's view, at the cost of declining a resize that would only fit by reusing
   the old pods' room); a volume checks only its growth; failing either rejects `reject_capacity`;
4. its configured pricing on the full new spec; above `max_price` it rejects `reject_price`, otherwise it accepts at
   that price.

Every answer names the request's digest.

**Applying a compute resize.** On `EventLeaseResized` the provider swaps the lease's reservation to the new size,
recomputes its scheduling parameters, and holds the reservation as pending until the new pods are healthy. The tenant
then sends the manifest for the new size; the provider checks it against the deployment's new hash and the resized
group and redeploys in place, in the same namespace: each service's pods roll one at a time with no surge, with
requests and limits from the new resources and the new replica count. Services that mount a volume keep it.

- A resize that changes no resources (the price alone) leaves the reservation as it is.
- If no manifest for the new size arrives within `--lease-resize-manifest-timeout` (10 minutes by default, the
  chain's default request lifetime), the reservation follows the deployment that is running again: the provider stops
  holding the difference, so a later roll may not fit.
- At startup and every five minutes, the provider compares each active lease's group on chain with its reservation
  and applies any resize whose event it missed (a restart between the accept and the event included).

**Applying a volume or bucket resize.** The provider raises its volume record to the new size (it never lowers it),
and the storage operator grows the claim that holds the volume, parked or attached; the storage driver expands the
filesystem. For a bucket it raises the bucket's quota.

### Parameters, Events, Errors, and Queries

Parameters (`akash.market.v1beta5.Params`):

| Parameter              | Field | Default | Meaning                                         |
| ---------------------- | ----- | ------- | ----------------------------------------------- |
| `lease_resize_enabled` | 8     | `false` | the feature flag for new requests               |
| `lease_resize_ttl`     | 9     | 10m     | how long a request waits for an answer; > 0     |

Events (`akash.market.v1`):

| Event                       | Fields                                  |
| --------------------------- | --------------------------------------- |
| `EventLeaseResizeRequested` | `id`, `max_price`, `expires_at`, `digest` |
| `EventLeaseResized`         | `id`, `price`, `prev_price`, `digest`   |
| `EventLeaseResizeCanceled`  | `id`, `reason`, `digest`                |

Errors:

| Module         | Code | Error                        | Meaning                                              |
| -------------- | ---- | ---------------------------- | ---------------------------------------------------- |
| market         | 42   | `ErrLeaseResizeDisabled`     | `lease_resize_enabled` is off                        |
| market         | 43   | `ErrLeaseResizePending`      | the deployment already has an unexpired resize       |
| market         | 44   | `ErrLeaseResizeNotFound`     | no pending resize for this lease                     |
| market         | 45   | `ErrLeaseResizeExpired`      | the request has expired                              |
| market         | 46   | `ErrLeaseResizePriceTooHigh` | the accept's price is above `max_price`              |
| market         | 47   | `ErrInvalidLeaseResize`      | the request or accept is invalid                     |
| market         | 48   | `ErrLeaseResizeChanged`      | the answer names another request's digest            |
| deployment     | 33   | `ErrInvalidResize`           | the new resources break a rule in [What a Resize May Change](#what-a-resize-may-change) |

Query: `LeaseResizes` (`GET /akash/market/v1beta5/leases/resizes`) lists pending resizes, filtered by `owner`,
`provider`, or `owner` and `dseq`, with pagination. Expired records are listed until they are removed.

State: the pending resizes live in the market store under an unused prefix, `{0x17, 0x00}`, keyed by
`(owner, dseq)`; no new store key. Genesis carries them as `lease_resizes` (field 5), at most one per deployment.

CLI: `akash tx deployment resize [sdl-file]` compares the SDL with the chain, applies the rules locally, and sends
the request for the one group that changed (with the new manifest hash for a compute group);
`akash tx market lease resize-cancel`; `akash query market lease resizes --owner --dseq --provider`. For operators,
`akash tx market lease resize-accept --price` and `resize-reject --reason capacity|price|unsupported|other`, which
look up the digest themselves; the provider daemon answers automatically.

## Rationale

- **Consent, in two steps.** A resize changes what the provider runs and what the tenant pays, so both sign. The
  request and the answer reuse the shape of order, bid, and lease, and the provider's existing event-driven loop.
- **Not a re-order.** Re-ordering the group needs a new order (blocked while one is active) and a new bid, so a new
  lease ID, a new namespace, and lost data: close and redeploy under another name.
- **Not a provider-signed quote in one tenant transaction.** That saves a transaction but makes the chain verify a
  provider signature over an off-chain quote, and makes the provider hold capacity for the quote's whole validity.
- **One pending resize per deployment**, and deployment updates held while it waits, because one manifest hash
  covers every group.
- **Time-based expiry, lazy.** The provider and tenant reason in wall-clock time; lazy expiry needs no end-block work,
  and storage stays bounded at one record per deployment.
- **Settle, then switch.** Lazy settlement charges the current rate for every unsettled block, so the rate can only
  change right after a settle.
- **A conservative capacity check.** The provider's view still counts the old pods, so requiring the new size to fit
  beside them can never over-commit. Crediting the old pods back per node is left for later, because reservations
  don't record node placement.

## Backward Compatibility

- **Chain.** The four messages, the `LeaseResizes` query, two parameters, three events, and eight error codes are new;
  `MsgUpdateDeployment` additionally fails while a resize is pending. Old node binaries cannot decode the new messages,
  so this rides a coordinated upgrade. It ships dark: the upgrade seeds `lease_resize_enabled=false` and
  `lease_resize_ttl=10m`, and governance turns it on. No store migration: the pending resizes use an unused prefix in
  the market store and start empty.
- **Providers.** A provider that has not upgraded never answers, so its tenants' requests expire; no lease changes
  until a provider signs an accept.
- **Tenants and clients.** Nothing changes for a deployment that never resizes. `tx deployment update` keeps refusing
  group changes and points to `tx deployment resize`.

## Test Cases

Chain, end to end with real handlers (`node/tests/e2e/leaseresize_test.go`), each with exact escrow math where money
moves:

| Test                               | Covers                                                                  |
| ---------------------------------- | ----------------------------------------------------------------------- |
| ComputeHappyPath                   | a compute resize up; settled at the old rate through the accept block, the new rate after; the runway event |
| Gates                              | flag off; adding a GPU; one pending per deployment; price too high; wrong denom; wrong provider; an expired request, and replacing it |
| UpdateWaitsForPendingResize        | `MsgUpdateDeployment` refused while a resize is pending                 |
| RejectCancelClose                  | reject, cancel, and a lease close each clear the request                |
| VolumeGrow                         | a volume grows; a hash, a shrink, and a size above `max_volume_size` are refused |
| AttachedComputeKeepsAttachment     | a compute lease with an attached volume keeps its attachment            |
| AnswerNamesItsRequest              | cancel and re-request in one block; the old digest is refused           |
| AcceptAfterSameBlockWithdraw       | an accept in the block of a withdrawal settles correctly                |
| AdoptGrownVolume                   | adopting a grown volume requires its new size                           |
| ReplicatingVolumeKeepsItsSize      | a volume in a live replication cannot be resized, at request and again at accept |
| compute resize downward            | a shrink, with exact escrow and withdrawal amounts                      |
| bucket grow                        | a bucket grows through a lease resize                                   |
| reclaim mid-resize                 | `MsgLeaseStartReclaim` clears a pending resize                          |

Unit tests cover the change rules (21 cases, plus the frozen count of volume-mounting units), message validation,
expiry at its boundary, the digest, the query by deployment, the CLI's choice of target lease, `PaymentUpdateRate`
(old rate through the current block, then the new one; overdrawn, closed, wrong-denom, and zero-rate refusals; the
same-block fallback; exact runway values), and on the provider: the resize evaluator (accept with the digest; each
reject reason; volume headroom on the growth; bucket growth without an expandable class; expired, answered, and
replaced requests; catch-up at start and the periodic re-list), the reservation swap (price-only, already-deployed,
and manifest-timeout cases), the resync of missed resizes, the manifest refetch after a resize, and the storage
operator growing parked and attached claims and bucket quotas without ever shrinking them.

Live, on a local chain with three providers on one kind cluster:

| Resize                                                                   | Result                                       |
| ------------------------------------------------------------------------ | -------------------------------------------- |
| 0.5 CPU, 512 MiB memory, 512 MiB ephemeral, 1 replica -> 1 CPU, 1 GiB, 1 GiB, 2 replicas | accepted in 2 s; 25.20 -> 100.94 uact per block; both pods running at the new size 5 s after the request, same namespace |
| -> 0.25 CPU, 256 MiB, 256 MiB, 1 replica                                  | accepted in 2 s; -> 13.83 uact per block; the pod running at the new size 4 s after the request |
| -> 2 CPU, 2 GiB with a ceiling of 1 uact                                  | rejected `reject_price` (quote 100); lease, price, and pod unchanged |
| CPU 2 -> 2.5 -> 1 -> 1.5 from Console                                    | accepted; the pod rolled each time; cancel, decline, and re-apply work |
| 0.5 -> 1 CPU accepted by hand while the provider was stopped              | on restart the provider resynced the reservation with the missed resize; the new manifest rolled the pod to 1 CPU |
| a managed Postgres database: memory 1 -> 2 GiB and its volume 2 -> 3 GiB  | both leases resized on chain; the database back in about 20 s with its data |
| a provider configured to grow a class whose Retain twin cannot expand (local-path) | the class is dropped at startup with an error, and its growth is declined `reject_unsupported` |

The test cluster's storage classes are all local-path, which cannot expand a claim, so the volume rows above prove
the chain and provider flow, not a disk actually growing: growing a claim on an expandable class (Ceph RBD, for one)
is covered by the storage operator's unit tests only. GPU resizes are covered only by unit tests (no GPUs in the test
cluster).

## Reference Implementation

On the chalabi2 forks, branch `storage-market/active-versions` of
[chain-sdk](https://github.com/chalabi2/chain-sdk/tree/storage-market/active-versions),
[node](https://github.com/chalabi2/node/tree/storage-market/active-versions), and
[provider](https://github.com/chalabi2/provider/tree/storage-market/active-versions); the resize-only history is
branch `storage-market/lease-resize`. Both are stacked on the Decoupled Storage Market's implementation, and the
reference upgrade handler seeds both features, each behind its own flag. Compute resizing does not depend on the
storage market.

| Piece                     | Where                                                                                  |
| ------------------------- | -------------------------------------------------------------------------------------- |
| Change rules              | `chain-sdk/go/node/deployment/v1beta4/resize.go` (`GroupSpec.Resize`)                  |
| Messages, record, digest  | `chain-sdk/proto/node/akash/market/v1beta5/{leasemsg,leaseresize}.proto`, `go/node/market/v1beta5/leaseresize.go` |
| Handlers                  | `node/x/market/handler/resize.go`                                                      |
| Store and query           | `node/x/market/keeper/{resize.go,keys/resize.go,grpc_query.go}`                        |
| Escrow                    | `node/x/escrow/keeper/keeper.go` (`PaymentUpdateRate`)                                 |
| Deployment update hold    | `node/x/deployment/handler/server.go`                                                  |
| CLI                       | `chain-sdk/go/cli/{deployment_tx.go,market_tx.go,market_query.go}`                     |
| Provider admission        | `provider/bidengine/resize.go`                                                         |
| Reservation swap          | `provider/cluster/inventory.go`                                                        |
| Manifest refetch          | `provider/manifest/{manager.go,service.go}`                                            |
| Volume and bucket growth  | `provider/operator/storage/{operator.go,reconciler.go,bucket.go}`, `provider/cluster/kube/client_volume.go` |

Provider flags: `--volume-growable-classes` names the disk classes the provider can expand in place (empty by
default, which declines every disk growth; buckets need none; a class whose Retain StorageClass does not allow
expansion is dropped at startup), and `--lease-resize-manifest-timeout` (10 minutes)
bounds how long an accepted resize holds capacity for the tenant's new manifest. Price and capacity use the
provider's existing bid settings.

## Security Considerations

- **Consent and price.** No lease changes without the provider's signature, and the provider cannot charge more than
  the tenant's ceiling. The digest ties each answer to one request, so a quote can never be applied to a request it
  was not computed for.
- **Billing.** Settle-then-switch charges every block at the rate in force when it was produced; an overdrawn account
  cannot be resized. The new rate starts at the accept, before the new pods run: the provider holds the new size from
  then on, and the tenant decides when to roll by sending the manifest.
- **Manifest consistency.** One pending resize per deployment, and no deployment updates while it waits, keep the
  deployment's hash and its groups in step.
- **Capacity.** The provider's check never over-commits against its own view, but nothing is held between the check
  and the accept, and a racing deployment can take the room in between; the redeploy then cannot schedule.
- **Failure after accept.** There is no rollback. If the new pods never become healthy, the provider's monitor may
  close the lease as unstable, as it would after any failed redeploy.
- **Denial of service.** Requests are tenant-paid transactions; storage is one record per deployment, and expiry
  needs no end-block work.
- **Volumes.** A volume only grows, within the governance bound, never during a live replication, and adoption and
  replicas track its current size, so no copy is ever smaller than its source.

## Limitations and Future Work

- **Classic persistent storage cannot grow.** StatefulSet claim templates are immutable; growing them needs
  template-preserving updates and per-claim patching. Durable data that must grow belongs on a storage-market volume.
- **Fixed in v1:** endpoints, GPU models, adding or removing GPUs, storage classes, placement, volume references, and
  adding or removing services, all of which change scheduling or networking and are closer to a redeploy.
- **Brief restart.** Pods roll with no surge. Zero-downtime rollouts, and crediting the old pods back per node in the
  capacity check, are future work.
- **No provider-initiated resize** (for example, downsizing an over-provisioned lease).
- **Reference implementation gaps:** if a provider misses `EventLeaseResized` and the tenant's new manifest arrives
  before the provider's next five-minute resync, that deploy uses the old scheduling parameters, which matters only
  for a change in GPU count (interconnect). There is no provider kind e2e for resize yet.

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
