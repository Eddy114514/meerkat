# 02 — Restore the outer reactive cache after a nested recompute

**Depends on:** nothing.
**Reference:** `cli-test-fixes`, `Manager::recompute_def` in
`meerkat-lib/src/runtime/manager/mod.rs` (~line 830), and the unit test
`test_recompute_def_restores_an_active_reactive_cache`.

## Rationale

This is a pre-existing reentrancy bug in ordinary (non-transactional) reactive
propagation. It has nothing to do with transactions or read-your-own-writes; it
was found while working on PR #189 and should not wait for it.

`Manager::reactive_cache` is a transient `Option<HashMap<(Symbol, Symbol), Value>>`
holding the cross-service dependency values for **the def currently being
recomputed**, so that `MemberAccess` resolves from cache instead of issuing a
remote lookup. `recompute_def` installs it on entry and, on `main`, clears it on
exit:

```rust
self.reactive_cache = Some(cache);
let result = eval(...).await;
self.reactive_cache = None;      // <-- bug
```

Evaluation can `await` a remote read. While it waits, `send_and_await_reply`
pumps network events, so an inbound `Update` can re-enter `handle_update` and
therefore `recompute_def` **while an outer recompute is suspended**. The inner
call then clears the cache the outer call installed. When the outer recompute
resumes, its member accesses miss the cache and fall through to `lookup`,
resolving against whatever state exists at that moment rather than the
dependency snapshot the recompute was started with.

## Specification

`recompute_def` must save the previous value of `self.reactive_cache` and
restore it, rather than clearing:

```rust
let outer_cache = self.reactive_cache.take();
self.reactive_cache = Some(cache);
let result = eval(...).await;
self.reactive_cache = outer_cache;
```

The restore must happen on **both** the success and the error path — the
existing code already evaluates into a `result` binding before matching on it,
so restoring immediately after the `eval` is sufficient. Do not introduce an
early `return` between the install and the restore.

## Tests

**Illustrating test:** `test_recompute_def_restores_an_active_reactive_cache`
(unit test in `manager/mod.rs` on the reference branch).

It installs a sentinel cache as if an outer recompute were in progress, runs a
`recompute_def`, and asserts `manager.reactive_cache` still holds the sentinel
afterwards. On `main` it holds `None` and fails.

This is the right primary test: fast, and exactly on the mechanism. It does
assert on a sentinel rather than on a program-visible outcome, so an optional
second test is described below.

### Optional: drive the reentrancy end to end

Worth adding if the sentinel assertion feels too close to the implementation.
Roughly 80 lines given the existing stand-in-peer helpers; use the technique
from `participant_commit_order_test.rs`, which makes an ordering observable
instead of timing-dependent by having a bare network peer control the sequence.

Setup: a def `d = a.x + b.y` on the node under test, with `b.y` warm in
`dep_cache` and `a.x` not. Trigger a propagation of `d`.

When the stand-in peer receives the `LookupRequest` for `a.x`, it sends **two**
messages: first an `Update` targeting a different def on the node under test,
then the `LookupResponse`. Both are queued before the node reads either, so the
pump handles the `Update` inside the await window without any sleeps.

Keep the def targeted by that `Update` **purely local**, so the inner recompute
completes without awaiting. A remote dependency there opens a second window and
the ordering stops being obvious.

Assert either way round:

- **Request count** (cleanest): with the fix, no second `LookupRequest` for
  `b.y` is ever sent, because `b.y` is still served from the restored cache.
  Without the fix, one is.
- **Value**: have the peer serve a different `b.y` on a fresh read than the one
  it pushed into `dep_cache`. With the fix the def uses the cached value;
  without, the freshly read one.

### Why there is no `.mkt` coverage

Recorded so this does not get re-derived. The trigger needs an inbound `Update`
to land inside the await window of an outer recompute's remote read. That
window is narrow but not rare: `dispatch_network_events` is called only from
inside `send_and_await_reply` (`manager/mod.rs:1211` and `:1252`), so a node
handles inbound messages *only* while waiting on an outbound call, which is
exactly when an outer recompute is suspended.

`.mkt` gives no lever to sequence that -- no sleep, no barrier, no control over
when a peer sends -- so such a test would be waiting on luck and a failure
would be indistinguishable from a flake.

There is also nothing clean to assert. Clearing the cache makes the outer
recompute's *remaining* member accesses fall through to `lookup`, so the def is
computed from a mix of snapshot and live values. That is a glitch -- an
inconsistent combination -- rather than a crisply stale value, and the live
value is arguably the fresher one. `.mkt` asserts on values, so there is no
natural assertion. (The one sharper symptom: if the fall-through read fails,
the eval errors, `propagate` logs and returns `false`, and the def silently
keeps its old value where the cache would have let it succeed. Arranging an
unreachable peer at that exact instant from `.mkt` is no easier.)

## Notes

- The reference branch applies the same save/restore to `recompute_def_in_txn`.
  That function is part of the eager-propagation mechanism we are not taking;
  the pattern returns in task 12, which reuses that function's body. Only fix
  `recompute_def` here.
- Extend the doc comment to explain *why* it is a save/restore and not a clear.
  The reference branch's comment is good; reuse it. Without the explanation this
  reads like a pointless complication and will be "simplified" back into a bug.
