---
aep: 87
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

4. **[Chain-Enforced Attachment](./README.md#attachment)** -- Attaching a volume to a compute deployment is gated at
   bid time (only the volume's current lessee may bid) and re-verified at lease time. A market attachment index bounds
   concurrent attachments (read-write-once in v1), and a **cascade-detach invariant** guarantees a volume lease never
   closes while a compute lease is still attached and writing.

5. **[Migration via Reclamation](./README.md#migration)** -- Volume migration reuses the lease reclamation primitive: a
   chain-enforced `min_volume_reclamation_window` parameter makes the migration window a protocol floor, the market's
   automatic re-order loop is the re-listing, and the destination provider pulls a checksummed, resumable snapshot
   stream from the source with authorization derived from chain state on both ends.

6. **[Replication](./README.md#replication)** -- A replica is its own volume deployment linked to its primary by a
   typed `replica_of` reference: its own lease, its own payment stream. The primary's export duty is declared in its
   group spec and therefore priced into the primary's bid, not an unpaid convention. The default transfer driver
   requires no Ceph cluster peering.

7. **[Exhaustion, Retention, and Adoption](./README.md#economics)** -- Escrow overdraw remains terminal on-chain, but
   exhaustion is made deterministic and survivable: a projected-depletion event at every settlement, a bounded on-chain
   retention window after close, and a chain-gated **adoption** flow that re-binds retained data to a fresh volume
   deployment, restricted to the provider actually holding it.

8. **[Phased Rollout](./README.md#phased-rollout)** -- One consensus upgrade ships everything dark behind a
   `volume_orders_enabled` parameter (default `false`); a governance parameter change activates the market; migration
   and replication follow as provider-only releases.

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
provides exactly the "grace window before teardown" primitive migration needs; AEP-75 multi-depositor escrow provides
funding flexibility for long-lived storage accounts; and AEP-86's `persistent_storage` capability flag provides the
provider-verification hook. This AEP composes those primitives rather than duplicating them: the alternative -- a
dedicated `x/storage` module -- was evaluated and rejected because it re-implements the majority of the market's state
machines as a permanently doubled audit surface while still needing every compute-market change for the attach path.

## Specification

### Design Invariants

Two invariants run through every section and every rollout phase:

1. **No deployed binary -- old or new -- ever deletes volume data as a side effect.** Volume persistent volumes are
   created with a `Retain` reclaim policy at provision time (safety is a property of the stored object, not of
   whichever binary later deletes a namespace), volume data lives outside lease namespaces, old provider binaries
   cannot decode volume orders at all, and exactly one code path -- the storage operator's garbage collection,
   driven by a chain-computable deadline -- destroys data.

2. **No split-brain.** A volume lease can never close -- by tenant close, provider reclaim, or escrow exhaustion --
   while a compute lease is still attached and writing: every volume-close path cascade-detaches (force-closes the
   attached compute leases in the same state transition) before the volume lease closes.

### Volume as a Storage-Only Deployment

A volume is a deployment with exactly one group whose group spec:

- carries a `VolumePolicy` message -- the typed discriminator; its presence marks the group as a volume group;
- declares CPU, memory, and GPU as present-but-zero, and exactly one persistent storage entry with `count: 1`.

Volume identity is the ordinary `(owner, dseq, gseq=1)` group identity. The durable data contract -- size, class, and
`VolumePolicy{vid, reclaim, retention, max_attachments, max_replicas}` -- lives on the group record, written once at
creation and persisting across every order, bid, lease, and attachment, readable even after the group closes.

`VolumePolicy` carries:

| Field             | Meaning                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------ |
| `vid`             | Tenant-chosen data-continuity label (DNS-label grammar); unique per owner among live volumes |
| `reclaim`         | `retain` or `delete` -- what happens to data after the volume lease closes                 |
| `retention`       | Post-close window during which the provider must keep the data adoptable; bounded by a governance parameter; known and priced at bid time |
| `max_attachments` | Bound on concurrent attached compute leases; validation forces `1` (read-write-once) in v1 |
| `max_replicas`    | How many replica volumes may reference this volume; a bid-time-known, priced export duty   |
| `adopt`           | When set, this volume adopts a dead retained volume (same owner, same `vid`)               |
| `replica_of`      | When set, this volume is a replica of the named volume (same owner)                        |

**Zero new message types.** The complete flow rides existing messages:

| Operation                              | Message (existing)                                          |
| -------------------------------------- | ----------------------------------------------------------- |
| Create volume / adopt / create replica | `MsgCreateDeployment` (storage-only group)                  |
| Fund / top up                          | `MsgCreateDeployment` deposit, `MsgDepositDeployment` (including AEP-75 grants) |
| Provider offer                         | `MsgCreateBid`                                              |
| Accept                                 | `MsgCreateLease`                                            |
| Close volume (tenant)                  | `MsgCloseDeployment` (routes through reclamation; rejected while attached) |
| Migrate / evict (provider)             | `MsgLeaseStartReclaim` then `MsgCloseBid`                   |
| Settle / detect exhaustion             | `MsgWithdrawLease`                                          |
| Attach                                 | Compute `MsgCreateDeployment` whose resources reference the volume |

A storage-only deployment has no services; its manifest is a canonical empty group whose hash is deterministic,
version-pinned, and treated with the same rigor as the manifest hash itself.

### Volume Identity (`vid`)

The `vid` is the data's name; the deployment sequence is merely its current market address. An on-chain
`(owner, vid) -> GroupID` index is maintained on volume create, close, and adoption, and enforces at creation time:

- A `vid` mapping to a **live** volume rejects a new volume under the same name.
- A `vid` mapping to a **retained** dead volume (closed, within its retention window) rejects a new volume under the
  same name **unless** the new volume's `adopt` reference names exactly that dead volume. A mistargeted redeploy can
  never silently mint an empty volume under an existing data identity.
- An expired or absent `vid` permits a fresh create.

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
                          |  provider bids (storage-only path)
                          v  tenant MsgCreateLease
        +------------  Lease active  ---------------------------+
        |  PROVISIONED: volume exists on provider, parked       |
        +--+-------------------------^--------------------------+
           | compute lease with      | compute lease closes
           | volume reference        | (any reason) -> detached
           v                         |
        ATTACHED (RWO: exactly one) -+
           |
           | volume lease close, ANY path -- all routes:
           |   1. reclamation window (>= min_volume_reclamation_window;
           |      skipped only for the exhaustion cascade)
           |   2. CASCADE-DETACH: attached compute leases force-closed
           |   3. volume lease closed with a volume close reason
           v
        RETAINED  -- group closed; provider holds data; deadline is
           |         chain-computable: lease close time + retention
           |
           +-- adoption within the window -> new volume deployment,
           |   same vid, bids gated to the retaining provider
           |
           +-- migration: group stays open -> automatic re-order ->
           |   new provider wins -> pulls snapshot from source
           |
           +-- window expires, no adoption or migration
                          v
        RELEASED -- provider garbage collection deletes the data.
                    THE ONLY DATA-DESTRUCTION PATH.
```

The compute-side lifecycle is untouched: an attaching compute deployment is ordinary; closing it fires the normal
cascade, which never reaches the volume -- different deployment, different escrow account.

### Market Integration

Volume orders flow through `CreateOrder -> CreateBid -> CreateLease` unmodified. The new market state is two additive
bid-time checks and one attachment index:

**Volume-group bids.** A bid on a volume order must offer storage quantities exactly equal to the order's request
(no under- or over-offer), and must commit a reclamation window at least `min_volume_reclamation_window` -- the
provider's on-chain, bonded retain-through-migration commitment. Adoption orders additionally gate bids to the provider
of the dead volume's most recently closed lease, determined purely from chain state.

**Attachment.** A compute group references volumes by typed `(owner, dseq, gseq, name)` reference on its resource
units. Referenced volumes contribute *zero* storage quantity to the compute group -- the size and class contract is the
volume's, so attached storage is never double-counted in compute inventory or pricing. At bid time, per reference, the
chain requires: the referenced group exists and is a volume group; the reference's owner equals the order's owner (no
cross-tenant attach); an **active lease** exists on the volume group and its provider equals the bidder -- the
chain-enforced colocation contract; and the volume's attachment count is below `max_attachments`. Because colocation
reads *current* state on every bid, it survives the automatic re-order loop and volume migration with zero extra code:
after a migration, attach bids automatically gate to the new provider. Colocation and the attachment bound are
re-verified at `CreateLease`, which then records the attachment and emits a typed attach event.

**Cascade-detach.** A volume lease never reaches `closed` while the attachment index holds entries for its group. On
every volume-close path -- reclaim completion, tenant close after its reclamation window, or escrow exhaustion -- the
attached compute leases are force-closed first with a distinct `volume_detach` reason (their own deployments survive
and re-order), and only then does the volume lease close with its volume reason. Work is bounded: at most
`max_attachments` (one, in v1) extra lease closures per volume.

**Tenant close routes through reclamation.** Closing a volume is never instantaneous: the lease enters reclamation for
at least the chain floor, giving migration a guaranteed window in both close directions. Closing a volume deployment
while a compute lease is attached is rejected outright -- a mounted database never loses its backing lease to a tenant
fat-finger; tenants close compute first. (The escrow-exhaustion cascade cannot be blocked; it cascade-detaches
instead.)

New vocabulary is additive only: volume close reasons in the established reason ranges, typed attach/detach/adopted
events, and a `volume_orders_enabled` + `min_volume_reclamation_window` parameter pair. Existing order, bid, lease, and
payment identity types are untouched.

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
      retention: 168h          # post-close adoption window (<= chain param)
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

**A compute deployment attaches by reference:**

```yaml
version: "2.2"

volumes:
  myapp-pgdata:
    external:
      dseq: 1234567            # or resolved from vid by the CLI

services:
  db:
    image: postgres:16
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
volume is an error; `count > 1` with a read-write-once reference is rejected. Mixed files (volumes and services
together) are client-side sugar: the CLI creates the volume deployment first, waits for its lease, then creates the
compute deployment -- on-chain there are always two deployments. AEP-15 per-lease storage syntax is unchanged and
remains the default.

CLI sugar papers over the "a disk is a deployment" UX leakage: `akash tx volume create|close|migrate|adopt` and
`akash query volume list|status --vid`, the latter resolving the `vid` index and surfacing lease state, attachment,
escrow runway, and the retention deadline.

### Provider Responsibilities

Participation is opt-in: a provider advertises `capabilities/storage/volumes` and installs a storage operator.
Non-participating providers are unaffected -- volume orders are new wire shapes their binaries cannot decode, and the
capability requirement filters non-participating upgraded providers.

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
- **One garbage-collection path.** Data is destroyed only when block time passes the chain-computable deadline
  (lease close time + retention) and no export or adoption is in flight. Never by lease-closed event inference.
- **Inventory honesty.** Volume bytes are durable allocations: bid-time gating reads live per-class capacity,
  durable reservations are never thin-provisioned or released on lease close (only on garbage collection), and
  parked or retained volumes are never double-counted as free capacity.

### Migration

Migration reuses the lease reclamation primitive verbatim -- no separate migration-window mechanism exists:

1. The provider starts reclaim on the volume lease (eviction), or the tenant closes with a migrate reason. The lease
   enters reclamation for at least the chain-enforced floor; the source provider's garbage collection freezes.
2. The market's automatic re-order loop re-lists the still-open volume group; a new provider wins.
3. The destination pulls a checksummed, resumable snapshot stream from the source over mutually authenticated TLS
   rooted in on-chain provider certificates. **Authorization derives from chain state on both ends**: the exporter
   serves only the provider holding the new active lease on this volume group; the importer accepts only from the
   provider of the volume's closed or reclaiming lease. No self-asserted identity anywhere.
4. At close, cascade-detach force-closes any attached compute leases; their deployments re-order and their new attach
   bids gate to the destination provider automatically.

The default transfer driver is snapshot stream export -- it works between mutually untrusted providers with no Ceph
cluster peering, no WAN-exposed storage daemons, and no storage credentials handed to a competitor. Ceph-native
mirroring is an opt-in driver for provider pairs that explicitly trust each other, with scoped credentials.

### Replication

A replica is its own volume deployment carrying a `replica_of` reference to its primary -- its own order, lease, escrow
account, and payment stream, on a different provider. This makes replication paid work on both sides via existing
machinery: the replica's provider is paid by the replica lease, and the primary's export duty is declared through
`max_replicas` in its group spec, so it was priced into the primary's bid rather than being an unpaid convention
between competitors.

Sync ships periodic snapshot diffs from the primary's provider to replica providers over the same authenticated
transfer service; lag is observable via a status query and surfaced by the CLI. **The chain attests nothing about
replica consistency in this AEP** -- promotion is tenant-driven: redeploy compute pointing at the replica's reference
(orchestrated by `akash volume promote`). Application-level replication remains fully supported and needs no provider
machinery at all.

### Economics

- **Accounts and payments.** One deployment-scoped escrow account per volume (the single-group invariant makes the
  account synonymous with the volume), created, funded, and topped up by the existing deployment paths, including
  AEP-75 multi-depositor grants. One payment stream per volume lease. Replicas are separate accounts and streams.
  Compute and volume runways are independent by construction.
- **Pricing.** Bids price the storage-only group through the existing per-class storage offer pricing. Retention
  length, reclamation window, and replica count are bid inputs folded into the single per-block rate; the chain
  validates exact size equality, the price ceiling, and the reclamation floor. IOPS and quality-of-service are
  off-chain pricing inputs, not chain-validated dimensions.
- **Deterministic runway.** Every settlement of an account with open payments emits a projected-depletion event --
  tenants and tooling compute the real exhaustion deadline, and providers withdraw on schedule *because* withdrawal is
  the exhaustion detector.
- **Exhaustion is terminal but survivable.** A missed top-up closes the volume lease through the cascade (with
  cascade-detach and a distinct unfunded reason). The provider then retains the data for the bounded, on-chain,
  priced-at-bid-time retention window.
- **Adoption.** Within the retention window, the tenant creates a new volume deployment whose `adopt` reference names
  the dead volume: `vid`-gated at creation, provider-gated at bid (only the provider actually holding the data may
  bid), re-bound by the operator. Data identity (`vid`) survives bankruptcy; market identity (dseq) does not -- the
  `vid` index makes re-resolution a query, not archaeology.
- **The trust framing, stated plainly.** The *reclamation window* is the guarantee -- chain-enforced and bonded by a
  still-open, still-paying lease. The *retention window* after close is a bounded, known, priced best-effort tail --
  never open-ended courtesy.

Escrow grace ("top up to save your database" on-chain) is deliberately deferred: it modifies the most
consensus-sensitive code in the stack. Its required shape is pre-specified in the design of record so a follow-up AEP
can add it without rework.

## Phased Rollout

Standing invariants across all phases: no deployed binary ever deletes volume data as a side effect, and release
sequencing is SDK tag, then node release, then provider release.

**Phase 0 -- hygiene and dormant plumbing (no consensus change).** Reconcile the staged proto version directories with
what the network actually runs; SDL v2.2 parser and version-gate fix; canonical-empty-manifest hashing with golden
tests; CLI sugar. Provider-side: storage operator, volume custom resource, retain storage classes, attach path,
bid path, transfer service -- all shipped dormant (no volume orders can exist on chain).

**Phase 1 -- consensus upgrade U1 (feature off; the only consensus upgrade).** Activates the new deployment and market
proto versions: storage-only validation, `VolumePolicy` and volume references, the bid gates, the attachment and `vid`
indexes (empty at genesis), the cascade-detach close routine, close-through-reclamation, close reasons and typed
events, the runway event, an escrow-hooks hardening fix, and store migrations -- with `volume_orders_enabled=false`.
The network upgrades with zero behavior change; every new path is unreachable behind the flag. Upgrade CI additionally
runs the previous provider release against the upgraded node and verifies it serves existing leases and places no bids
on the new order shapes. Stated plainly: this is a hard fleet upgrade -- providers that skip it can serve existing
leases but cannot bid on any new orders until they upgrade, which is standard network-upgrade practice.

**Phase 2 -- governance flips `volume_orders_enabled=true`.** Delivers volume create/bid/lease with per-volume escrow,
chain-enforced attach colocation and single-attach, the chain-guaranteed reclamation floor, cascade-detach, runway
events, exhaustion-retention-adoption, SDL v2.2, and the one-command CLI. Consensus surface: none -- a parameter
change.

**Phase 3 -- migration and replication (provider releases; zero consensus changes).** The transfer service goes GA
with the stream-export driver; eviction runbooks; replica volumes and diff-shipping sync; migrate/promote
orchestration; the Ceph-native mirror driver as opt-in.

**Future AEPs (out of scope here).** Multi-attach (read-write-many) classes, volume resize, escrow grace,
chain-attested replica consistency and auto-promotion, and a dedicated storage query facade if UX demands one -- the
identity tuples chosen here let a future facade alias them without breaking anything.

## Security Considerations

- **Data destruction.** Exactly one code path destroys volume data: the provider operator's garbage collection, past
  the chain-computable deadline (lease close time + retention), frozen while an export or adoption is in flight.
  Retain-at-provision, out-of-lease-namespace data, and order shapes old binaries cannot decode make destruction via
  namespace teardown, lease-closed reflexes, old binaries, or daemon downgrades structurally impossible.
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
- **Data-plane authentication and authorization.** Mutual TLS rooted in on-chain provider certificate identities;
  authorization derived from chain state on both ends; snapshot-based source-immutable export with per-chunk and
  whole-image checksums. No storage-cluster daemons or pool-scoped credentials are exposed between untrusted providers
  by default; native mirroring is opt-in with scoped peers only. A compromised exporter can serve corrupt data --
  importers verify checksums, and application-level replication plus tenant-side encryption remain the paranoid
  options (encryption-at-rest guarantees are out of scope and documented as such).
- **Provider trust model, stated.** A provider can always destroy data it hosts; retention honesty is economic and
  reputational -- bounded, priced, and bonded through reclamation -- softened by replication (`max-replicas >= 1` is
  recommended and the CLI warns at zero). What this design removes is *protocol-caused* destruction. There are no
  cryptographic proofs of storage in v1.
- **Economic griefing and denial of service.** Volumes are deposit-backed deployments (existing spam economics); the
  attachment index is bounded by active leases; all new bid-time checks are constant-time reads, gas-accounted, with
  deterministic iteration; retention is capped by a governance parameter and priced; cascade-detach closes at most
  `max_attachments` leases per volume.
- **Consensus safety.** No new module, stores, hooks, or end-block logic; all new validation is version-gated and
  feature-flagged; existing identity types and their casts are untouched; the escrow-hooks fix converts a mid-write
  state-corruption path into a no-op with behavior-preservation tests; manifest-hash and group-spec determinism are
  preserved (new fields exist only under new versions; the canonical empty manifest is golden-tested); genesis
  export/import and consensus-version migrations ship with every new index and parameter.

## Limitations

- **No resize.** Group specs are immutable after creation; growing a volume means migrating to a larger one until a
  future group-update AEP. Inherited from the platform, not created here.
- **Overdraw is terminal on-chain.** A missed top-up closes the volume lease. The runway event, the retention window,
  and adoption compensate -- but market identity (dseq) does not survive bankruptcy, and attach references to the dead
  dseq must be re-resolved (a query, via the `vid` index).
- **Read-write-once only at launch.** `max_attachments` is validated to one; read-write-many is a later validation
  relaxation on an existing field, not a schema change.
- **Attach bids are effectively single-provider.** Colocation means the volume placement decision was the competitive
  moment; attach pricing has no competition. Accepted and surfaced in tooling.
- **Nothing self-heals without the tenant in v1.** Migration, adoption, and promotion all require the tenant to accept
  a bid or redeploy; auto-promotion needs chain-attested replica health -- a future AEP.
- **Explorer UX leakage.** "Create a disk" is a deployment message; explorers show volumes as deployments until they
  learn the `VolumePolicy` discriminator. Papered over with CLI sugar, accepted otherwise.
- **A hard fleet upgrade.** Providers that skip the node upgrade window cannot bid on any new orders until they
  upgrade.

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
