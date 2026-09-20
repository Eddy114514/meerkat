# 04 — Register and instantiate services correctly in the CLI

**Depends on:** 03 (the dependency-ordered `unified_ast` is what makes the
instantiation order correct).
**Reference:** `cli-test-fixes`, `meerkat/src/main.rs` — `local_service_names`,
`register_remote_services` (~line 522), and the instantiation block in
`run_client` (~line 1125). Commit `689aaf1` is the slug-collision half.

## Rationale

Three defects in how the CLI turns a parsed program plus `-i svc=url` flags into
live services.

**(i) Remote registration was lazy and therefore order-dependent.** Registering
a remote service only when its root `Stmt::Import` is reached fails two ways: an
*earlier* local import walks the whole unified AST and builds the remote service
as a phantom local copy; and a remote service reached only *transitively* has no
root `Stmt::Import` at all, so it is never registered — every read and every
action against it silently targets the phantom local copy instead of the owning
node.

**(ii) Imports were instantiated where their `Stmt::Import` appeared.** The
grammar allows `import` to appear *after* the service that uses it, and static
checks accept this because the unified AST is reordered (task 03). Creating each
import when its `Stmt::Import` is reached therefore builds `app` before the `dep`
it reads, failing with `ServiceNotFound`.

**(iii) `-i svc=url` was applied unconditionally.** `Manager::lookup` consults
`remote_services` **first**, before anything local. So `-i s2=/ip4/.../p2p/...`
against a program that itself declares `service s2` routes that service's reads
and writes to a peer and leaves the locally declared copy permanently
unreachable — silently. This is almost always a typo on the command line. Both
`run_server` and `run_client` had their own copy of the registration loop and
both had the defect.

## Specification

Add two free functions in `meerkat/src/main.rs`:

```rust
/// The names of the services this program declares itself.
fn local_service_names(prog: &[Stmt]) -> HashSet<Symbol>;

/// Apply the `-i svc=url` registrations to `manager`, skipping any name the
/// program declares itself.
fn register_remote_services(
    manager: &mut Manager,
    remote_url_map: &HashMap<String, String>,
    local: &HashSet<Symbol>,
);
```

`register_remote_services` must, for each `-i` entry:
- intern the name; if it is in `local`, **print a warning naming the flag and
  the service, and skip it**. Do not drop it silently — the user needs to know
  the flag had no effect;
- otherwise insert into `manager.remote_services` and report the registration.

Both `run_server` and `run_client` must call it. Neither may keep its own loop.

### `run_server` needs the unified AST first

The sequence below walks `manager.unified_ast`, and **in server mode that field
is never populated**. `run_client(full_ast, prog, ...)` takes both ASTs and
assigns `manager.unified_ast = full_ast` (`main.rs:997-1011`); `run_server(prog,
...)` takes a single `Vec<Stmt>`, constructs `Manager::new(interner)`, and never
assigns it. Step 3 would have nothing to walk and local imports would stay
uninstantiated.

Worse, the caller already discards the imports before `run_server` sees them:

```rust
let target_ast = if remote_url_map.is_empty() {
    node.unified_ast.clone()   // imports included
} else {
    prog                       // imports dropped whenever -i is used
};
```

So fix the plumbing as part of this task:

- Give `run_server` the same shape as `run_client`: take `full_ast` **and**
  `prog` as separate parameters, and assign `manager.unified_ast = full_ast`
  immediately after constructing the `Manager`.
- Delete the `target_ast` ternary at the call site and pass
  `node.unified_ast.clone()` and `prog` unconditionally. The ternary exists
  only because `run_server` had one slot for two different things, and it
  silently strips imports in exactly the configuration that needs them most.

`Node::on_manager_startup` (`node.rs:464`) and `run_client` both make this
assignment; the `Manager` built inside `run_server` is the only one that never
does. That is why the test harnesses and client mode work and the server does
not.

### Startup order

1. Compute `local_service_names(&prog)`.
2. `register_remote_services(...)` — **before any import is processed**.
3. Instantiate every locally resolved import by walking `manager.unified_ast` in
   order, filtering to `Stmt::Service` entries whose name is neither in
   `local_service_names` nor in `manager.remote_services`, skipping any already
   created.
4. Create the program's own services from `prog`, in program order.

`run_server` has no equivalent of step 3 at all, and no equivalent of step 1
either. Its two `&prog` loops print service URLs and create services
respectively; the `-i` registration between them is unconditional. So
`local_service_names(&prog)` is new code in the server path, not a value
already lying around to be reused — compute it before
`register_remote_services`, or the collision guard silently does nothing there.
Adding both is part of this task, not a follow-up.

## Tests

**Illustrating test:** `register_remote_services_skips_locally_declared_services`
— registers `-i s2=<url>` for a program declaring `service s2`, and asserts
`manager.remote_services` does not contain `s2`. Without the guard, a subsequent
`lookup` of an `s2` member routes to the peer.

Also: `local_service_names_covers_declarations_only` (non-`Stmt::Service`
statements must not contribute names).

Both are unit tests in `meerkat/src/main.rs` on the reference branch.

**New — the server path needs its own coverage, and a seam to test through.**
Both tests above exercise the registration helper, not startup, and the
reference branch has nothing covering `run_server`'s instantiation. That is
precisely the gap that let server mode ship with no import-instantiation step
at all.

`run_server` cannot be asserted against as it stands: it builds its `Manager`
locally, returns `Result<(), Box<dyn Error>>`, and then runs the event loop
forever. Extract the startup half into a helper that **returns the initialised
`Manager`** — everything from `Manager::new` through step 4, stopping before
the loop — and have `run_server` call it. That is a worthwhile change on its
own: it puts server and client startup on the same shape and makes the ordering
in steps 1-4 something a test can see.

Then assert that a program with a locally resolved import ends up with the
imported service in `manager.services`, **and** that it is still there when an
unrelated `-i` flag is present — the configuration where the `target_ast`
ternary used to strip the imports.

## Notes

- Keep the explanatory comments from the reference branch on the ordering in
  steps 2 and 3. Each encodes a bug that was hit in practice; without them the
  sequence looks arbitrary and will be reordered.
- `Manager::lookup` checking `remote_services` first is the behaviour that makes
  (iii) dangerous. Do not change it here — it is correct for genuinely remote
  services, and the fix belongs at registration time.
