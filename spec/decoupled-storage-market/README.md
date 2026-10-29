---
title: "Decoupled Storage Market"
author: Joseph Chalabi (@chalabi2)
status: Draft
type: Standard
category: Core
created: 2026-07-08
requires: 15
discussions-to: https://github.com/orgs/akash-network/discussions/372
roadmap: major
---

## Summary

This AEP defines:

1. **[Volume as a Storage-Only Deployment](./README.md#volume-as-a-storage-only-deployment)** -- A volume is an
   ordinary deployment whose single group carries a typed `VolumePolicy` discriminator. The entire existing market --
   orders, bids, leases, per-deployment escrow accounts, AEP-75 multi-depositor funding, lease reclamation, the closure
   cascade, and provider order discovery -- operates on volumes with no new chain module, no new store keys, no new
   message types, and no new escrow scope. "Storage survives compute" is not new survival machinery; it is the absence
   of coupling: a volume is its own deployment with its own escrow account, and nothing in the closure cascade crosses
   deployment boundaries.

2. **[Volume Identity (`vid`)](./README.md#volume-identity-vid)** -- A tenant-chosen data-continuity label, enforced
   unique per owner among live volumes by an on-chain index. The `vid` survives providers, migration, and adoption, and
   makes silently provisioning an empty volume under an existing data identity structurally impossible.

3. **[SDL v2.2 Volume Syntax](./README.md#sdl-syntax)** -- A top-level `volumes:` stanza (following the AEP-17
   `endpoints:` declare/reference pattern) that compiles to a storage-only group, and a service-level `volume:`
   reference that attaches an externally leased volume to a compute deployment.

4. **[Chain-Enforced Attachment](./README.md#market-integration)** -- Attaching a volume to a compute deployment is
   gated at bid time (only the volume's current lessee may bid) and re-verified at lease time. A market attachment index
   bounds concurrent attachments (read-write-once in v1), and a **cascade-detach invariant** guarantees a volume lease
   never closes while a compute lease is still attached and writing.

5. **[Buckets](./README.md#buckets)** -- A volume's `access` field makes it either a disk that one colocated compute
   lease mounts, or an S3-compatible bucket the provider serves over HTTPS to any client holding the owner's keys.
   Buckets are leased, paid, retained, adopted, and moved like disks, and the chain refuses to attach one.

6. **[Migration via Reclamation](./README.md#migration)** -- Volume migration reuses the lease reclamation primitive:
   a chain-enforced `min_volume_reclamation_window` parameter guarantees a minimum notice before any volume lease
   ends, the market's automatic re-order loop is the re-listing, and the new provider pulls the source's final copy (a
   checksummed, resumable stream) with authorization derived from chain state on both ends.

7. **[Replication](./README.md#replication)** -- A replica is its own volume deployment linked to its primary by a
   typed `replica_of` reference: its own lease, its own payment stream. The primary's export duty is declared in its
   group spec and therefore priced into the primary's bid, not an unpaid convention. The default transfer driver
   requires no Ceph cluster peering.

8. **[Exhaustion, Retention, and Adoption](./README.md#economics)** -- Escrow overdraw remains terminal on-chain, but
   exhaustion is made deterministic and survivable: a projected-depletion event at each settlement, a bounded on-chain
   retention window after close, and a chain-gated **adoption** flow that re-binds retained data to a fresh volume
   deployment, restricted to the provider actually holding it.

9. **[Phased Rollout](./README.md#phased-rollout)** -- One consensus upgrade ships the volume market dark behind a
   `volume_orders_enabled` parameter (default `false`); a governance parameter change activates it; moves and
   replication follow as provider-only releases.

## Motivation

Akash destroys data by construction today. A persistent volume claim is born inside a lease-derived namespace, the
provider's storage classes delete on reclaim, and closing a lease deletes the namespace -- the lease-closed event *is*
the data-destruction command. AEP-15 gave tenants persistence *within* a lease and explicitly deferred the rest:
"Future updates may explore data persistence across multiple leases, further expanding functionality." Nothing survives
the lease. Five years on, that deferred future is the single largest gap between Akash and the storage primitives every
public cloud tenant takes for granted.

The consequences are structural, not cosmetic:

- **Stateful workloads are second-class.** Databases, chain nodes, model checkpoints, and vector stores -- the
  workloads that most need decentralized compute -- cannot risk a platform where any lease closure, provider
  maintenance window, or missed escrow top-up erases the disk.
- **Storage and compute cannot be priced or scaled independently.** A tenant who needs 10 TB of durable data and a
  burst of GPU compute must couple both lifetimes into one lease, paying for the coupling in risk.
- **There is no migration story.** When a provider decommissions hardware or a tenant wants a better price, data does
  not move; it dies.

Community discussion ([akash-network/discussions#372](https://github.com/orgs/akash-network/discussions/372)) has
converged on the need for lease-decoupled storage, cross-provider durability, and a storage market with its own
economics. This AEP is the design of record for that capability.

Why now: the network's recent platform work makes a minimal design possible for the first time. Lease reclamation
provides exactly the "grace window before teardown" primitive migration needs, and AEP-75 multi-depositor escrow
provides funding flexibility for long-lived storage accounts. This AEP composes those primitives rather than
duplicating them: the alternative -- a dedicated `x/storage` module -- was evaluated and rejected because it
re-implements the majority of the market's state machines as a permanently doubled audit surface while still needing
every compute-market change for the attach path.

## Specification

### Design Invariants

Two invariants run through every section and every rollout phase:

1. **No deployed binary -- old or new -- ever deletes volume data as a side effect.** Volume persistent volumes are
   created with a `Retain` reclaim policy at provision time (safety is a property of the stored object, not of
   whichever binary later deletes a namespace), volume data lives outside lease namespaces, a provider binary without
   volume support can never win a volume order (the chain rejects any volume bid that does not carry the order's
   volume policy), and exactly one component destroys data: the storage operator's garbage collection, after the
   volume's retention deadline.

2. **No split-brain.** A volume lease can never close -- by tenant close, provider reclaim, or escrow exhaustion --
   while a compute lease is still attached and writing: every volume-close path cascade-detaches (force-closes the
   attached compute leases in the same state transition) before the volume lease closes.

### Volume as a Storage-Only Deployment

A volume is a deployment with exactly one group whose group spec:

- carries a `VolumePolicy` message -- the typed discriminator; its presence marks the group as a volume group;
- has a single resource unit with no endpoints that declares CPU, memory, and GPU as present-but-zero, and exactly one
  named persistent storage entry with a storage class and `count: 1`.

Volume identity is the ordinary `(owner, dseq, gseq=1)` group identity. The durable data contract -- size, class, and
`VolumePolicy{vid, reclaim, retention, max_attachments, max_replicas, access}` -- lives on the group record, set at
creation and persisting across every order, bid, lease, and attachment, readable even after the group closes. Only
the size can change afterwards, and only by growing through a lease resize, which the In-Place Lease Resize AEP specifies.

`VolumePolicy` carries:

| Field             | Meaning                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------ |
| `vid`             | Tenant-chosen data-continuity label (DNS-label grammar); unique per owner among live volumes |
| `reclaim`         | `retain` or `delete`: whether the provider keeps the data through `retention` after the volume lease closes, or releases it at close |
| `retention`       | Post-close window during which the provider keeps the data, for adoption or for a move's destination to copy; bounded by the `max_volume_retention` governance parameter (30 days by default); known and priced at bid time |
| `max_attachments` | Bound on concurrent attached compute leases: `1` for a disk (read-write-once), `0` for a bucket |
| `max_replicas`    | How many replica volumes may reference this volume; a bid-time-known, priced export duty; bounded by `max_volume_replicas`; `0` for a bucket |
| `adopt`           | When set, this volume adopts a dead retained volume (same owner, `vid`, and `access`; at least its size) |
| `replica_of`      | When set, this volume is a replica of the named disk volume (same owner; at least its size) |
| `access`          | `mount` (default): a disk that colocated compute leases attach. `bucket`: an S3-compatible bucket served over the network ([Buckets](./README.md#buckets)) |

**Zero new message types.** The complete flow rides existing messages:

| Operation                              | Message (existing)                                          |
| -------------------------------------- | ----------------------------------------------------------- |
| Create volume / adopt / create replica | `MsgCreateDeployment` (storage-only group)                  |
| Fund / top up                          | `MsgCreateDeployment` deposit, `MsgAccountDeposit` (including AEP-75 grants) |
| Provider offer                         | `MsgCreateBid`, carrying the order's `volume_policy`        |
| Accept                                 | `MsgCreateLease`                                            |
| Close volume (tenant)                  | `MsgCloseDeployment` or `MsgCloseGroup` (routes through reclamation; rejected while attached); a volume group cannot be paused or restarted |
| Move (tenant)                          | `MsgCloseLease` on the volume lease (routes through reclamation, then re-lists) |
| Move / evict (provider)                | `MsgLeaseStartReclaim` then `MsgCloseBid`                   |
| Settle / detect exhaustion             | `MsgWithdrawLease`                                          |
| Attach a disk                          | Compute `MsgCreateDeployment` whose resources reference the volume |

A storage-only deployment has no services. Its deployment hash is the hash of a canonical empty manifest (one group,
no services), deterministic, version-pinned, and golden-tested with the same rigor as the manifest hash itself;
providers never receive a manifest for a volume lease.

### Volume Identity (`vid`)

The `vid` is the data's name; the deployment sequence is merely its current market address. An on-chain
`(owner, vid) -> GroupID` index points at the volume that most recently took the name (it is written on every volume
create, adoption included); whether that volume is live, retained, or expired is read from its group and leases. At
creation time the index enforces:

- A `vid` mapping to a **live** volume (group open) rejects a new volume under the same name.
- A `vid` mapping to a **retained** dead volume (closed or unfunded, within its retention window counted from its last
  lease's close time) rejects a new volume under the same name **unless** the new volume's `adopt` reference names
  exactly that dead volume. A mistargeted redeploy can never silently mint an empty volume under an existing data
  identity.
- When the volume holding the name never had a lease (an adoption closed before it was leased), it holds no data: the
  rules above apply to the volume it was adopting, so the name stays retained, and a new adoption must name that
  volume.
- An expired or absent `vid` permits a fresh create.

The retention window applies to every closed volume whatever its `reclaim` policy: a `delete` volume holds its `vid`
through its retention as well, although its provider released the data at close.

The `vid` is a typed field, never a storage attribute: attributes flow into provider attribute matching, and a
tenant-unique value there would make the order unbiddable by every provider.

### Volume Lifecycle

No new state enums. The composite state is derived from existing deployment, group, and lease states plus the
attachment index:

```
        MsgCreateDeployment (SDL volumes: -> storage-only group)
                          |
                          v
        Deployment active / Group open / Order open
                          |  provider bids with the order's volume policy
                          v  tenant MsgCreateLease
        +------------  Lease active  ---------------------------+
        |  PROVISIONED: volume exists on provider, parked       |
        +--+-------------------------^--------------------------+
           | compute lease with      | compute lease closes
           | volume reference        | (any reason) -> detached
           | (disks only)            |
           v                         |
        ATTACHED (RWO: exactly one) -+
           |
           | volume lease close, every path:
           |   1. reclamation window (>= min_volume_reclamation_window;
           |      skipped only for the exhaustion cascade)
           |   2. CASCADE-DETACH: attached compute leases force-closed
           |   3. volume lease closed with a volume close reason
           |
           +-- move (tenant MsgCloseLease, provider evict): the group
           |   stays open -> automatic re-order -> new provider wins
           |   -> copies the source's final data -> PROVISIONED there
           |
           v
        RETAINED  -- tenant close or escrow exhaustion ends the group;
           |         provider holds the data until close + retention
           |
           +-- adoption within the window -> new volume deployment,
           |   same vid, bids gated to the retaining provider
           |
           +-- window expires, no adoption
                          v
        RELEASED -- provider garbage collection deletes the data
                    (right at close for reclaim: delete).
                    THE ONLY DATA-DESTRUCTION PATH.
```

After a move, the source keeps its copy until the close plus the volume's retention, then releases it the same way.

The compute-side lifecycle is untouched: an attaching compute deployment is ordinary; closing it fires the normal
cascade, which never reaches the volume -- different deployment, different escrow account.

### Market Integration

Volume orders flow through `CreateOrder -> CreateBid -> CreateLease` unmodified. The new market state is a set of
additive bid-time checks and one attachment index:

**Volume-group bids.** A bid on a volume order must carry the order's volume policy (`MsgCreateBid.volume_policy`,
which a provider without volume support never sets), offer storage quantities exactly equal to the order's request
(no under- or over-offer), and commit a reclamation window of at least `min_volume_reclamation_window` (24 hours by
default): the provider's on-chain, bonded notice before it can end the lease. A bid carrying a volume policy on any
other order is rejected. Adoption orders additionally gate bids to the provider of the dead volume's most recently
closed lease, determined purely from chain state.

**Attachment.** A compute group references volumes by typed `(owner, dseq, gseq, name)` reference on its resource
units. Referenced volumes contribute *zero* storage quantity to the compute group -- the size and class contract is the
volume's, so attached storage is never double-counted in compute inventory or pricing. At bid time, per reference, the
chain requires: the referenced group exists and is a disk volume group (a bucket is never attachable) whose storage
entry has the referenced name; the reference's owner equals the order's owner (no cross-tenant attach); an **active**
lease (not one in reclamation) exists on the volume group and its provider equals the bidder -- the chain-enforced
colocation contract; and the volume's attachment count is below `max_attachments`. Because colocation reads *current*
state on every bid, it survives the automatic re-order loop and volume migration with zero extra code: after a
migration, attach bids automatically gate to the new provider. Every attach check is re-run at `CreateLease`, which
then records the attachment and emits a typed attach event.

**Cascade-detach.** A volume lease never reaches `closed` while the attachment index holds entries for its group. On
every volume-close path -- reclaim completion, tenant close after its reclamation window, or escrow exhaustion -- the
attached compute leases are force-closed first with a distinct `volume_detach` reason (their own deployments survive
and re-order), and only then does the volume lease close with its volume reason. Work is bounded: at most
`max_attachments` (one, in v1) extra lease closures per volume.

**Tenant close routes through reclamation.** Closing a volume is never instantaneous: the first close puts the lease
into reclamation for its window (at least the chain floor), and a repeated close before the deadline fails. When the
window ends, either the tenant's second close or the provider's close of the lease completes it, with the same end
state; a tenant close after the provider's succeeds and changes nothing. `MsgCloseLease` on the volume lease is a move
(`volume_migrate`): the group stays open and re-lists. `MsgCloseDeployment` and `MsgCloseGroup` end the volume
(`volume_closed_retain`: the deployment, or only the group, closes as the tenant asked) and are rejected outright while
a compute lease is attached -- a mounted database never loses its backing lease to a tenant fat-finger; tenants close
compute first. Moving an attached volume is allowed: its compute is detached when the window ends. (The
escrow-exhaustion cascade cannot be blocked; it cascade-detaches instead.) A volume group cannot be paused or
restarted: a paused volume would hold its name while its data aged out, and a restarted one could be won by a provider
that holds none of its data.

**Provider reclaim.** A provider ends a volume lease by starting reclaim. Whatever reason it gives, the chain records
`volume_evict` and re-lists the group when the window ends, exactly as a tenant move does, so the next provider has a
source to copy from.

New vocabulary is additive only: volume close reasons in the established reason ranges, typed attach/detach/adopted
events, the bid's `volume_policy`, the lease's close time, volume error codes, and the parameters
`volume_orders_enabled`, `min_volume_reclamation_window`, `max_volume_size`, `max_volume_retention`, and
`max_volume_replicas`. Existing order, bid, lease, and payment identity types are untouched.

### Buckets

A volume's `access` field selects how its data is reached. `mount` (the default) is the disk described above.
`bucket` is an S3-compatible bucket the provider serves over HTTPS: any deployment on any provider, or any client
anywhere, reaches it with access keys that only the volume's owner can obtain. A bucket is leased, paid, retained,
adopted, and moved like a disk; only the data path differs.

|                | Disk (`access: mount`)                               | Bucket (`access: bucket`)                     |
| -------------- | ---------------------------------------------------- | --------------------------------------------- |
| Reached by     | mounting into a compute lease on the volume's provider | the S3 API over HTTPS, from anywhere         |
| Attachments    | one compute lease at a time (`max_attachments: 1`)   | none (`max_attachments: 0`); any number of clients with keys |
| Replicas       | `max_replicas`, `replica_of`                         | none                                          |
| A move carries | the volume's image or file tree                      | every object with its metadata                |
| Suited to      | databases, anything that needs a filesystem          | datasets, model weights, media, backups, shared files |

**Chain rules**, on top of every volume rule above:

- A bucket group has `max_attachments = 0`, `max_replicas = 0`, and no `replica_of`, and no replica may name a bucket.
  Its storage entry keeps the disk shape; the class names the storage the bucket's objects sit on, which the provider
  prices and reserves.
- A volume reference to a bucket is refused at `CreateBid` and `CreateLease`, so a bucket is never mounted.
- An adoption keeps the dead volume's `access`, and the bid's `volume_policy` must equal the order's, so a bucket order
  cannot be bid as a disk or the reverse.
- The SDL requires `capabilities/storage/buckets: true` instead of `capabilities/storage/volumes: true`, so only
  providers that serve buckets see bucket orders.

**Provider rules.** Each bucket is its own backend user with a quota equal to the leased size; writes past it fail.
The provider creates the bucket, denies its deletion by policy, and blocks public access. The owner mints and revokes
keys through the provider gateway on the bucket's own lease: `read` or `readwrite`, at most 20 per bucket, the secret
returned once and never stored by the provider. Keys cannot change the bucket's policy, ACLs, or public-access block,
or create or delete buckets; sharing is by presigned URL. Closing the lease stops every key. After retention the
bucket, its objects, and its backend user are deleted. An adoption revokes every old key before the bucket reopens.

**Moving a bucket** follows [Migration](./README.md#migration), with objects in place of an image. Through the
reclamation window the bucket keeps serving. Once its lease closes, the source seals it (revokes every key, so nothing
writes again) and serves it to the destination as final. The destination refuses new keys until the copy lands; then
the owner mints new keys. Each provider serves its own endpoint, so clients reconnect after a move. A move carries
object bytes, content headers, `Cache-Control`, `Expires`, and user metadata; it does not carry keys, object tags,
ACLs, versions, or incomplete multipart uploads.

**Pricing.** A bucket's capacity is priced like a disk of its class. Traffic is not metered on chain: providers price
expected traffic into the bid and may rate-limit per bucket.

### SDL Syntax

SDL v2.2 adds a top-level `volumes:` stanza on the AEP-17 `endpoints:` declare/reference/error-if-unused pattern.

**A volume is its own deployment:**

```yaml
version: "2.2"

volumes:
  myapp-pgdata:                # key == vid
    size: 200Gi
    class: beta3               # existing class whitelist; ram rejected
    lifecycle:
      reclaim: retain          # retain | delete
      retention: 168h          # post-close window (<= chain param)
    max-replicas: 2            # export duty priced into the bid (0 = none)

profiles:
  placement:
    us-west:
      attributes: { region: us-west }
      pricing:
        myapp-pgdata: { denom: uact, amount: 150 }

deployment:
  myapp-pgdata:
    us-west: { profile: myapp-pgdata, count: 1 }
```

**A bucket is a volume with `access: bucket`:**

```yaml
version: "2.2"

volumes:
  media:
    size: 50Gi
    class: beta3               # the storage the objects sit on
    access: bucket             # mount (default) | bucket
    lifecycle:
      reclaim: retain
      retention: 168h

profiles:
  placement:
    us-west:
      attributes: { region: us-west }
      pricing:
        media: { denom: uact, amount: 150 }

deployment:
  media:
    us-west: { profile: media, count: 1 }
```

**A compute deployment attaches a disk by reference:**

```yaml
version: "2.2"

volumes:
  myapp-pgdata:                # the volume's vid
    external:
      dseq: 1234567            # the volume deployment; owner defaults to the signer

services:
  db:
    image: postgres:16
    expose:
      - port: 5432
        to:
          - global: true
    params:
      storage:
        data:
          mount: /var/lib/postgresql/data
          volume: myapp-pgdata

profiles:
  compute:
    db:
      resources:
        cpu: { units: 2 }
        memory: { size: 4Gi }
        storage: []            # attached volume contributes NO local storage
  placement:
    anywhere:
      pricing: { db: { denom: uact, amount: 800 } }

deployment:
  db: { anywhere: { profile: db, count: 1 } }
```

A `volume:` storage param excludes local `size`/`class`/`attributes`; a mount is required; a declared-but-unreferenced
volume is an error; `count > 1` with a read-write-once reference is rejected; one volume cannot be referenced by two
services, and services sharing a profile must reference the same volumes. A file either declares volumes or
references them: on-chain there are always two deployments, and the volume is created first. A reference to a bucket
fails on chain. AEP-15 per-lease storage syntax is unchanged and remains the default.

The CLI wraps the deployment commands for volumes: `akash tx volume create` and `akash tx volume close`, and
`akash query volume list|status`, which show a volume's groups, escrow account, and leases. A move is
`akash tx market lease close` on the volume lease; adoption and replicas are SDL fields (`adopt`, `replica-of`) on a
new volume. `akash tx market bid create` copies the order's volume policy into the bid.

### Provider Responsibilities

Participation is opt-in: a provider advertises `capabilities/storage/volumes` (disks) or `capabilities/storage/buckets`
(buckets) and runs the storage operator; buckets also need an S3 backend. Non-participating providers are unaffected:
the capability requirement keeps volume orders from matching them, and the chain rejects any volume bid that does not
carry the order's volume policy, which their binaries never set.

The provider-side contract, at specification level (mechanics in
[IMPLEMENTATION.md](./IMPLEMENTATION.md#5-provider-changes)):

- **Retain at provision.** Every persistent volume backing a volume group is created with a `Retain` reclaim policy,
  from retain-variant storage classes installed alongside the existing AEP-15 classes (which are untouched). Namespace
  teardown, lease-closed reflexes, daemon downgrades, and operator bugs can at worst release the volume -- the data
  survives.
- **Data lives outside lease namespaces.** The volume's durable cluster object is cluster-scoped; attaching binds it
  into the compute lease's namespace by direct reference, and the volume is never left in an unclaimed state a cluster
  binder could hand to another tenant.
- **A restart-survivable record.** The operator maintains a per-volume custom resource recording owner, `vid`,
  on-chain identity, class, size, retention, and phase; all state is re-derivable from cluster objects plus chain
  queries.
- **One garbage-collection path.** Data is destroyed only after its deadline: the lease's close time on chain plus
  the volume's retention, with a short margin past it so an adoption in the deadline's last block never races the
  collection (a `delete` volume is released at close). A volume that moved is held until the move's copy has landed
  (see [Migration](./README.md#migration)). Never directly on a lease-closed event.
- **Inventory honesty.** Volume bytes are durable allocations: bid-time gating reads live per-class capacity, durable
  reservations are never thin-provisioned or released on lease close, and parked, retained, and moving volumes count
  as used capacity until they are released.
- **Transfer service.** A participating provider serves its volumes to the provider that wins them, and to replica
  providers, over the transfer service on its gateway's gRPC port, under the authorization rules in
  [Migration](./README.md#migration).
- **Buckets.** A provider that serves buckets runs an S3-compatible backend and enforces the rules in
  [Buckets](./README.md#buckets).

### Migration

Migration reuses the lease reclamation primitive verbatim; no separate migration-window mechanism exists. The
reclamation window is the notice period, and the copy runs after the source lease closes, while the source holds the
volume for the move:

1. The provider starts reclaim on the volume lease (`volume_evict`), or the tenant closes it (`MsgCloseLease`,
   `volume_migrate`). The lease enters reclamation for its window, at least the chain floor. The volume keeps serving
   its workload.
2. When the window ends, the lease closes (the provider's `MsgCloseBid`, or the tenant's close after the deadline):
   cascade-detach force-closes any attached compute lease, and the market's automatic re-order loop re-lists the
   still-open volume group. The source holds the data for the move, whatever the volume's retention (below).
3. The tenant accepts a bid from a new provider. The destination provisions the volume as importing and pulls the
   source's data once the source is final (its lease closed and nothing mounting the volume): a checksummed, resumable
   image or file stream for a disk, every object with its metadata for a bucket. The transfer runs over mutually
   authenticated TLS rooted in on-chain provider certificates. **Authorization derives from chain state on both
   ends**: the exporter serves only a provider holding an active lease on the same volume group; the importer accepts
   only a provider that held this volume (same owner and `vid`) before, newest first, falling back to an older holder
   when a newer one's own copy never finished arriving. No self-asserted identity anywhere.
4. Compute attach bids gate to the destination as soon as it holds the lease; the destination declines to attach the
   volume until the copy lands and the volume is parked. If the copy keeps failing, the destination starts its own
   reclaim and the order re-lists again, and the source keeps holding the data for the next destination.

**The move hold.** After a move closes the source lease, the source keeps the volume until a complete final copy has
gone to the provider holding the new lease, plus a short margin for that provider to verify it, and never before the
close plus the volume's retention. The hold is bounded: the source releases the volume no later than the close plus
the longer of its retention and the move hold (7 days by default, the time a re-listed volume order stays open for
bids). Every volume can therefore move, retention or not; retention only adds the adoption tail. The workload is down
from the close until the copy lands and its compute is redeployed at the destination.

The default transfer driver is stream export: a snapshot of the volume's image (a file tree for non-RBD classes)
streamed by the provider itself. It works between mutually untrusted providers with no Ceph cluster peering, no
WAN-exposed storage daemons, and no storage credentials handed to a competitor. The driver is pluggable; Ceph-native
mirroring for provider pairs that explicitly trust each other fits the same interface.

### Replication

A replica is its own volume deployment carrying a `replica_of` reference to its primary -- its own order, lease, escrow
account, and payment stream, on a different provider. This makes replication paid work on both sides via existing
machinery: the replica's provider is paid by the replica lease, and the primary's export duty is declared through
`max_replicas` in its group spec, so it was priced into the primary's bid rather than being an unpaid convention
between competitors.

The replica's provider pulls from whichever provider holds the primary's active lease (re-read from chain every round,
so a replica follows its primary through moves) over the same authenticated transfer service: a whole copy first, then
periodic copies on an interval it sets (snapshot diffs for RBD volumes). The replica cannot be attached until its first
whole copy has landed, and a round never writes while compute holds the volume. The primary's provider serves a replica
only to the provider holding that replica's lease, and never pulls from its own replica; a provider declines to hold a
replica of a primary it holds itself. Lag is reported on the replica's lease status. **The chain attests nothing about
replica consistency in this AEP** -- promotion is tenant-driven: redeploy compute pointing at the replica's reference.
Once compute attaches to a replica, it stops syncing for good, so a promoted replica is never overwritten. Buckets have
no replicas. Application-level replication remains fully supported and needs no provider machinery at all.

### Economics

- **Accounts and payments.** One deployment-scoped escrow account per volume (the single-group invariant makes the
  account synonymous with the volume), created, funded, and topped up by the existing deployment paths, including
  AEP-75 multi-depositor grants. One payment stream per volume lease. Replicas are separate accounts and streams.
  Compute and volume runways are independent by construction.
- **Pricing.** Bids price the storage-only group through the existing per-class storage offer pricing. Retention
  length, reclamation window, and replica count are bid inputs folded into the single per-block rate; the chain
  validates exact size equality, the price ceiling, and the reclamation floor. IOPS and quality-of-service are
  off-chain pricing inputs, not chain-validated dimensions.
- **Deterministic runway.** Each settlement that advances an account with open payments and leaves it funded emits a
  projected-depletion event -- tenants and tooling compute the real exhaustion deadline, and providers withdraw on
  schedule *because* withdrawal is the exhaustion detector.
- **Exhaustion is terminal but survivable.** A missed top-up closes the volume lease through the cascade (with
  cascade-detach and a distinct unfunded reason). The provider then retains the data for the bounded, on-chain,
  priced-at-bid-time retention window.
- **Adoption.** Within the retention window, the tenant creates a new volume deployment whose `adopt` reference names
  the dead volume (same `vid` and `access`, at least the same size): `vid`-gated at creation, provider-gated at bid
  (only the provider actually holding the data may bid), re-bound by the operator. Data identity (`vid`) survives
  bankruptcy; market identity (dseq) does not, and the owner finds the data again by its `vid`, which is recorded on
  the volume's group.
- **The trust framing, stated plainly.** The *reclamation window* is the guarantee: chain-enforced notice, bonded by a
  still-open, still-paying lease, through which the volume keeps serving. After the close, the provider owes two
  bounded, unpaid tails: the *move hold*, until a moving volume's copy has landed (at most 7 days by default), and the
  *retention window*, the known, priced tail that covers adoption. Neither is open-ended courtesy, and both are
  best-effort promises rather than bonded ones.

Escrow grace ("top up to save your database" on-chain) is deliberately deferred: it modifies the most
consensus-sensitive code in the stack. Its required shape is pre-specified in the design of record so a follow-up AEP
can add it without rework.

## Phased Rollout

Standing invariants across all phases: no deployed binary ever deletes volume data as a side effect, and release
sequencing is SDK tag, then node release, then provider release.

**Phase 0 -- hygiene and dormant plumbing (no consensus change).** SDL v2.2 parser and version-gate fix;
canonical-empty-manifest hashing with golden tests; the volume CLI. Provider-side: storage operator, volume custom
resource, retain storage classes, attach path, bid path, transfer service, bucket backend -- all shipped dormant (no
volume orders can exist on chain).

**Phase 1 -- consensus upgrade U1 (feature off; the only consensus upgrade).** Adds the storage-market fields to the
active deployment and market API versions and the stable types: storage-only validation, `VolumePolicy` (with
`access`) and volume references, the bid's volume policy, the bid gates, the attachment and `vid` indexes (rebuilt
from state at the upgrade and at genesis import, so no store migration), the cascade-detach close routine,
close-through-reclamation, close reasons, typed events and error codes, the runway event, the lease close time, and an
escrow-hooks hardening fix -- with `volume_orders_enabled=false`. With the flag off no volume can be created and no
volume bid is accepted. A few changes are live from the upgrade: every lease records its close time, settlements emit
the runway event, the escrow-hooks fix applies, a bid carrying a volume policy on a non-volume order is rejected, and a
compute deployment that references a volume is accepted but can draw no bid. Providers that do not upgrade keep
serving existing leases and bidding on compute orders; they cannot win volume orders, because their bids carry no
volume policy.

**Phase 2 -- governance flips `volume_orders_enabled=true`.** Delivers disk and bucket create/bid/lease with
per-volume escrow, chain-enforced attach colocation and single-attach, the chain-guaranteed reclamation floor,
cascade-detach, runway events, exhaustion-retention-adoption, SDL v2.2, and the volume CLI. Consensus surface: none --
a parameter change.

**Phase 3 -- moves and replication (provider releases; zero consensus changes).** The transfer service with the
stream-export driver, moving disks and buckets between providers; replica sync; eviction runbooks; promote tooling;
native mirroring as an opt-in driver.

**Out of scope here.** Lease resizing, including growing a volume or bucket in place, is specified in the In-Place Lease Resize AEP.
Future AEPs: multi-attach (read-write-many) classes, escrow grace, chain-attested replica consistency and
auto-promotion, and a storage query service (`vid` and attachment lookups) if UX demands one -- the identity tuples
chosen here let a future facade alias them without breaking anything.

## Security Considerations

- **Data destruction.** Only the storage operator's garbage collection destroys a volume's data (a disk's persistent
  volume and image; a bucket, its objects, and its backend user), and only after its deadline: the lease's close time
  on chain plus retention and a short margin, or, for a volume that moved, once its copy has landed (bounded by the
  move hold). Retain-at-provision and out-of-lease-namespace data make destruction via namespace teardown,
  lease-closed reflexes, or daemon downgrades structurally impossible, and a provider binary without volume support
  never holds a volume, because it cannot win one. An import replaces only the destination's own older copy of the
  same volume.
- **Volume re-bind race.** The volume's cluster object is always handed directly from holder to target and back --
  never left unclaimed where the cluster's binder could give a data-bearing volume to an arbitrary tenant -- and a
  released volume is never adopted for a different owner.
- **Split-brain.** The cascade-detach invariant guarantees no compute lease writes to a volume after the market has
  re-homed or unfunded it. Read-write-once is double-enforced: the on-chain attachment index on every close path, and
  a single pre-bound claim in-cluster.
- **Cross-tenant attach.** The volume reference's owner must equal the attaching order's owner at both bid and lease
  time. Cross-tenant sharing is out of scope (it needs an authorization design).
- **Empty-volume footgun.** Per-owner `vid` uniqueness plus the retained-`vid` adoption requirement make silently
  provisioning a fresh volume under an existing data identity a hard chain error, never a silent success.
- **Bucket access.** Keys come only from the provider gateway, to the bucket lease's on-chain owner (mutual TLS with
  the owner's certificate, or a JWT signed by the account key, with the bucket scopes), per bucket, read-only or
  read-write, revocable one by one; the secret is shown once and the provider stores none. Keys cannot make a bucket
  public, change its policy, or create or delete buckets, and objects uploaded with a public ACL are still refused to
  anonymous readers. Closing the lease stops every key; an adoption revokes every old key. The storage provider can
  read the data, as with any hosted storage; clients that need confidentiality encrypt before upload.
- **Data-plane authentication and authorization.** Mutual TLS rooted in on-chain provider certificate identities,
  with the requester bound to its client certificate; authorization derived from chain state on both ends; a move
  copies a final source only; transfers resume and carry per-chunk and whole-image checksums (per-chunk and
  per-object for buckets). No storage-cluster daemons or pool-scoped credentials are exposed between untrusted
  providers, and nothing a peer sends reaches a shell on the exporter's storage tools. A compromised exporter
  can serve corrupt data -- importers verify checksums, and application-level replication plus tenant-side encryption
  remain the paranoid options (encryption-at-rest guarantees are out of scope and documented as such).
- **Provider trust model, stated.** A provider can always destroy data it hosts; retention honesty is economic and
  reputational -- bounded, priced, and bonded through reclamation -- softened by replication (`max-replicas >= 1` is
  recommended and the CLI warns at zero). A move's copy relies on the source honoring the move hold after the lease
  closes. What this design removes is *protocol-caused* destruction. There are no cryptographic proofs of storage in
  v1.
- **Economic griefing and denial of service.** Volumes are deposit-backed deployments (existing spam economics); the
  attachment index is bounded by active leases; the new bid-time checks are gas-accounted with deterministic
  iteration, and the colocation and adoption checks read only the volume group's own orders and bids; retention is
  capped by a governance parameter and priced; cascade-detach closes at most `max_attachments` leases per volume.
  Bucket traffic is not metered on chain; providers rate-limit per bucket.
- **Consensus safety.** No new module, store keys, hooks, or end-block logic; the new fields are additive on the
  active API versions, and every new volume path sits behind the feature flag (the changes live from the upgrade are
  listed under Phased Rollout); existing identity types and their casts are untouched; the escrow-hooks fix converts a
  mid-write state-corruption path into a no-op with behavior-preservation tests; manifest-hash and group-spec
  determinism are preserved (the new manifest field is omitted when empty, so existing hashes are unchanged, and the
  canonical empty manifest is golden-tested); the new indexes are rebuilt from state at genesis import and at the
  upgrade, so they need no genesis fields or store migrations.

## Limitations

- **Resize is specified separately.** Growing a volume or bucket in place is part of the In-Place Lease Resize AEP.
- **Moves copy the whole volume, after the close.** The copy runs once the source lease has closed, so the workload is
  down from the close until the copy lands, and the source holds the data unpaid for at most the move hold.
- **Overdraw is terminal on-chain.** A missed top-up closes the volume lease. The runway event, the retention window,
  and adoption compensate -- but market identity (dseq) does not survive bankruptcy, and attach references to the dead
  dseq must be re-resolved by `vid`.
- **Read-write-once only at launch.** `max_attachments` is validated to one for disks (buckets take none);
  read-write-many is a later validation relaxation on an existing field, not a schema change.
- **Attach bids are effectively single-provider.** Colocation means the volume placement decision was the competitive
  moment; attach pricing has no competition. Accepted and surfaced in tooling.
- **A moved bucket has a new endpoint and keys.** Each provider serves its own S3 endpoint, so clients reconnect with
  new keys after a move.
- **Nothing self-heals without the tenant in v1.** Migration, adoption, and promotion all require the tenant to accept
  a bid or redeploy; auto-promotion needs chain-attested replica health -- a future AEP.
- **Explorer UX leakage.** "Create a disk" is a deployment message; explorers show volumes as deployments until they
  learn the `VolumePolicy` discriminator. Papered over by the volume CLI, accepted otherwise.

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
