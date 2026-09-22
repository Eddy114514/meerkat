# 08 — Report commit failures instead of swallowing them

**Depends on:** 07.
**Reference:** `cli-test-fixes` — `struct ParticipantCommit`
(`manager/mod.rs` ~line 122), the commit block in `execute_action_with_txn`
(~line 2595), `commit_participant` (~line 2768), and
`meerkat-lib/tests/commit_failure_test.rs`.

## Rationale

The result of committing a participant was discarded with `let _ = ...`. A
participant that refuses the commit, or never answers, left the originator
returning `Ok(())`.

The user-visible symptom: the CLI prints `@test(...) passed` for a transaction
only part of which is actually committed.

## Specification

Introduce a return type carrying **two independent facts**:

```rust
/// What a participant's `Commit` left behind: the locks it released, and any
/// failure forwarding that commit to the nodes below it.
#[derive(Debug, Default)]
pub struct ParticipantCommit {
    /// Locks released by this commit, to wake anything parked on them.
    pub freed: HashSet<WaitKey>,
    /// Failure forwarding `Commit` to a sub-participant, reported to the
    /// originator once the local commit has finished.
    pub forward_error: Option<EvalError>,
}
```

`commit_participant` returns this instead of `HashSet<WaitKey>`.

**Do not fold these into a `Result`.** The caller needs both on the failing
path. The local commit is irreversible once the writes are stored, so a
forwarding failure is something to report upward — not a reason to skip waking
whatever was parked on the locks this commit just freed. A `Result` forces the
caller to choose one, and it will choose wrong.

In `execute_action_with_txn`, the commit loop records the **first** failure and
keeps going (every participant still needs its `Commit`), then still runs
`propagate_committed_writes`, and finally returns the recorded error in place of
`Ok(())`.

### The originator's side of a refused commit

`send_commit` reconstructs an `EvalError` from `CommitResponse`'s `error`
string (~line 2643 on `main`). It is the fourth such site in `manager/mod.rs`;
task 05 routed the other three through `Manager::remote_error`, and both that
task's spec and the reference branch missed this one. It only starts to matter
here, because `execute_action_with_txn` discards its result today (`let _ =
self.send_commit(...)`, ~line 2166) — the line this task removes.

Decide explicitly rather than by default, and record the reasoning in the PR:

- **Route it through `remote_error`** and a `WaitDieAbort` raised beneath a
  participant's commit is preserved, and becomes retryable like any other.
- **Leave it flattened** to `LocalDispatchFailed` and a refused commit stays
  terminal.

The default is to leave it flattened, for this task's own reason about #191:
the writes above the failure are already durable, so committing is not the
safe, idempotent operation an action is. Preserving the variant would feed a
partially committed transaction back into the wait-die retry budget. If that
reasoning turns out to be wrong it is a one-line change — which is the point
of having `remote_error` in one place.

## Tests

`meerkat-lib/tests/commit_failure_test.rs` (2 tests).

**Illustrating test:** `test_originator_surfaces_a_participant_commit_failure` —
a stand-in participant refuses the commit; the originator's `execute_action`
must return `Err`. On `main` it returns `Ok` and the CLI reports the test as
passing.

Also: `test_participant_reports_a_failed_commit_forward_and_still_propagates`,
which pins the "both facts, independently" property — a middle node whose
downward forward fails must still store, still propagate, still free its locks,
*and* report the error.

## Notes

- This does **not** make the transaction atomic under a refused commit. The
  writes above the failure are already durable. That is tracked separately as
  issue #191. This task makes the failure *visible*, which is a precondition for
  fixing it, not the fix. Say so in the PR description so it is not mistaken for
  a completeness claim.

- **Typed wire errors: considered here, still deferred.** Raised as a nitpick
  on #199. `remote_error` classifies a remote failure by matching the `Display`
  prefix `WaitDieAbort` writes, so a human-readable string is load-bearing as a
  protocol field. This task is the natural place to re-examine that — it is the
  first to make a *second* kind of `EvalError` cross the wire and be acted on
  rather than printed. It is still not worth acting on:
  - All six error-carrying `MeerkatMessage` variants are `String` /
    `Option<String>`, filled by ~19 producers with `e.to_string()`. Typing one
    makes it inconsistent with the other five; typing all six is ~32 sites plus
    a wire-side error enum, since `EvalError` is not `Serialize` and
    `WaitOn(WaitKey)` cannot trivially become so. Neither belongs in the same
    diff as a commit-reporting fix.
  - The version-skew argument for it does not apply. There is no version
    negotiation beyond libp2p's exact-match `/meerkat/1.0.0` and no
    `#[serde(default)]` anywhere, so adding a typed field is itself a breaking
    struct change: it relocates the mixed-version hazard from "misread as
    terminal" to "message fails to decode" rather than removing it.

  Revisit if a third consumer appears, or when protocol versioning does.
