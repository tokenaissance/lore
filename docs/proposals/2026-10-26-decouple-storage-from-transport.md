---
lep: 2026-10-26-decouple-storage-from-transport
title: Decouple lore-storage from lore-transport behind a remote-source port
authors:
  - mengmengmengqiang
status: Draft
created: 2026-10-26
updated: 2026-10-26
---

# Decouple `lore-storage` from `lore-transport` behind a remote-source port

## Summary

`lore-storage` depends on `lore-transport` for exactly two items: `StorageSession` and
`ProtocolError`. `StorageSession` appears in 56 public signature positions as
`Option<Arc<StorageSession>>`, the "optional remote" argument that the read, write, defragment and
fragment-engine entry points take.

The system design (§6.3) says an application that needs only the storage primitives can use Lore as
a storage system and ignore version control. It cannot currently ignore the wire: the smallest
standalone storage use still links `lore-transport`, and with it `quinn`, `rustls`, `rcgen`, `tonic`
and `prost`.

This proposal introduces one port in `lore-storage`:

```rust
// lore-storage/src/remote.rs
#[async_trait]
pub trait RemoteFragmentSource: Send + Sync {
    async fn get(&self, address: &Address) -> Result<(Fragment, Bytes), StorageError>;
    async fn get_priority(&self, address: &Address) -> Result<(Fragment, Bytes), StorageError>;
    async fn get_resolved(&self, key: &Hash, context: &Context, flags: u32)
        -> Result<(Hash, Fragment, Bytes), StorageError>;
    async fn put(&self, address: Address, fragment: Fragment, payload: Option<Bytes>)
        -> Result<(), StorageError>;
    async fn verify(&self, address: &Address, heal: bool) -> Result<VerifyResult, StorageError>;
    async fn can_copy_from(&self, partition: Partition) -> bool;
    async fn invalidate(&self);
}
```

`Option<Arc<StorageSession>>` becomes `Option<Arc<dyn RemoteFragmentSource>>`, and
`protocol_error_to_storage` moves out of `lore-storage` into the adapter that implements the port.
`lore-storage` stops depending on `lore-transport`. No wire format, on-disk format or behaviour
changes.

## Motivation

Measured at `8185adf7`:

| Item | Count |
| --- | --- |
| Files in `lore-storage/src` importing `lore-transport` | 5 (`read.rs`, `write.rs`, `defragment.rs`, `fragment_engine.rs`, `error.rs`) |
| Distinct `lore_transport::` items used | 2 (`StorageSession`, `ProtocolError`) |
| `Option<Arc<StorageSession>>` signature occurrences | 56 (`read.rs` 19, `write.rs` 19, `defragment.rs` 13, `fragment_engine.rs` 5) |
| `session.*` call sites in `lore-storage` | 7 (`get`, `get_priority`, `get_resolved`, `put`, `verify`, `can_copy_from`, `invalidate`) |

Three consequences:

1. **The standalone-storage claim is weaker than the document says.** §6.3 offers the storage
   subsystem as an independent product ("to deduplicate large data sets, to address content
   cryptographically, or to ship binary blobs through a multi-tenant service"). Every one of those
   uses still compiles the QUIC/TLS stack, because the remote argument's type comes from it.
2. **The build graph contradicts the layer diagram.** §5.1 draws storage below version control with
   the wire *beside* both. In the manifest, `lore-storage` sits above `lore-transport`. Every
   transport change rebuilds storage, and a storage-only consumer inherits transport's dependency
   tree.
3. **Local-only tests carry a wire type.** Unit tests that exercise local reads or defragmentation
   must name a session type, or pass `None` through a signature that exists only to carry the remote.

This is a layering improvement, not a bug report: passing a session works, and the code is careful
about it (`fetch_remote`, `heal_on_receipt`, single-flight, `SlowDown` backoff). The question is
whether the coupling should be a *type* dependency or a *port*.

## Goals

- `cargo tree -p lore-storage --depth 1` no longer lists `lore-transport`.
- No change to: wire format, on-disk format, error semantics seen by callers, or the behaviour of
  fetch/heal/retry.
- Call-site migration is mechanical: wrap the session once where it is created.

## Non-goals

- Changing the storage protocol or `StorageSession`'s own API.
- Removing remote fetch from the storage subsystem (see Option C).
- Touching `lore-revision`'s dependency on `lore-proto`, which is a separate proposal.
- Making `lore-storage` usable without a remote at runtime; the remote stays part of the subsystem.

## Design

### Port placement

Two placements are viable.

**A. New crate `lore-storage-remote` (recommended).** Depends on `lore-storage` and
`lore-transport`; implements `RemoteFragmentSource for StorageSession` and holds the
`ProtocolError` → `StorageError` mapping. `lore-transport` stays protocol-only, `lore-storage`
stays transport-free, and neither gains a dependency on the other. Cost: one crate.

**B. Implement in `lore-transport`.** No new crate, but `lore-transport` must depend on
`lore-storage` for `Fragment`, `VerifyResult` and `Partition`. That reverses the current edge rather
than removing it: storage would no longer depend on transport, but transport would depend on
storage's content model, which the crate name does not advertise.

Both keep the dependency graph acyclic: after step 1, `lore-storage`'s remaining dependencies are
`lore-base`, `lore-io`, `lore-telemetry`, `lore-error-set` and `lore-macro`.

### Errors

The port returns `StorageError`, so the adapter maps protocol errors. `protocol_error_to_storage`
moves verbatim to the adapter; its four branches (`AddressNotFound`, `Disconnected`, `SlowDown`,
`NotConnected`) and the address it preserves stay identical. `error.rs` then has no `lore-transport`
use.

### Object safety

`Arc<dyn RemoteFragmentSource>` with `#[async_trait]`, which `lore-storage` already depends on. The
call is one vtable dispatch per remote operation, behind an `Arc` and a network round trip; the
local path never touches the trait object.

## Change list

| # | File | Change | Rough size |
| --- | --- | --- | --- |
| 1 | `lore-storage/src/remote.rs` | new: the port, its doc comments, re-export from `lib.rs` | ~40 lines |
| 2 | `lore-storage/src/lib.rs` | `pub mod remote;` + re-exports | 3 lines |
| 3 | `lore-storage/src/read.rs` | 19 signatures; 4 call sites (`get`, `get_priority`, `get_resolved`, `verify`, `invalidate`) | mechanical |
| 4 | `lore-storage/src/write.rs` | 19 signatures; 2 call sites (`put`, `can_copy_from`) | mechanical |
| 5 | `lore-storage/src/defragment.rs` | 13 signatures | mechanical |
| 6 | `lore-storage/src/fragment_engine.rs` | 5 signatures | mechanical |
| 7 | `lore-storage/src/error.rs` | remove `protocol_error_to_storage` (moves to the adapter) | −18 lines |
| 8 | `lore-storage/Cargo.toml` | drop `lore-transport` | −1 line |
| 9 | `lore-storage-remote/src/lib.rs` | new: impl + error mapping + a `RemoteSession` wrapper | ~90 lines |
| 10 | `lore`, `lore-revision`, `lore-server` call sites | wrap the session once: `session.map(RemoteSession::new)` | ~6 sites |

## Migration steps

Each step is independently reviewable and keeps the tree green.

1. **Add without switching.** Land `remote.rs`, the new crate and the impl. Old signatures stay.
   No behaviour change, nothing depends on the port yet.
2. **Switch `lore-storage`.** Replace the 56 signatures and the 7 call sites; delete the mapping
   from `error.rs`; drop the `lore-transport` dependency. Compiler-guided; no logic edits.
3. **Update callers.** `lore`, `lore-revision` and `lore-server` wrap the session where it is
   resolved (`immutable::resolve_session`, `lore/src/storage/store.rs`), not at every call.
4. **Guard the edge.** Add a check that fails if `lore-storage` ever depends on `lore-transport`
   again — `cargo tree -p lore-storage --depth 1` output, or a manifest assertion in the existing
   verify scripts.

## Validation

- `cargo build`, `cargo +nightly fmt --all`, `cargo clippy --all-targets -- -D warnings --no-deps`.
- `cargo tree -p lore-storage --depth 1` shows no `lore-transport`.
- Existing `lore-storage` unit and conformance tests unchanged and green.
- Error mapping: a `SlowDown` from the session still surfaces as `StorageError::SlowDown`, and a
  missing address still surfaces as `AddressNotFound` with its address. Add one test per branch at
  the adapter.
- Live smoke on the demo server: `lore clone`, edit, `lore push`, and one `lore sync` that
  produces a `SlowDown`-free success. This exercises `get`, `put`, `verify`, `can_copy_from`.
- Since the mapping is moved rather than rewritten, the strongest check is a diff review of
  `protocol_error_to_storage`: it must be textually identical apart from its module path.

## Alternatives considered

- **Cargo feature gate.** Rejected: ADR-0013 records that features are not used to separate
  dependencies here, and a feature would still couple the two crates' build graph when enabled.
- **Generic `S: RemoteFragmentSource`.** Rejected: it infects every public signature and
  monomorphizes the read/write paths per source type, for no runtime benefit.
- **Option C — move remote fetch/heal/single-flight out of `lore-storage` entirely.** This is the
  better end state: storage keeps content and local persistence, and an adapter crate owns "local
  miss → remote fetch → verify → heal → cache". It is a much larger change and would move
  `read_flight` and the `SlowDown` handling with it. The port in this proposal is the prerequisite
  that makes Option C mechanical, so it is a follow-up, not a competitor.
- **Do nothing.** Defensible if §6.3's standalone-storage promise is not a product goal. The
  proposal is worth taking only if that promise is real.

## Open questions

1. Does the standalone-storage use case (external consumer of the storage API) actually exist, or
   is §6.3 aspirational? If it is aspirational, this is a nice-to-have.
2. Should the port live in `lore-storage`, or in a separate `lore-storage-api` crate that both
   storage and its consumers depend on? The second avoids growing `lore-storage`'s own surface.
3. `lore-revision` and `lore` call `session.copy`, `put_resolved` and `query` directly. Those are
   outside the 7 call sites today. If the port later has to cover them, it should be named for
   "remote storage operations", not only fragments.
