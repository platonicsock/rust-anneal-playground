# Anneal Playground Performance Notes

## Goal

Measure setup latency separately from steady-state verification cost.

Current Docker image build measurements:

```text
current_successful_build_ms: 426586
current_successful_build_human: 7m 6.586s
immediate_cached_rebuild_ms: 1270
immediate_cached_rebuild_human: 1.270s
```

The first number is the meaningful current compiler image build duration. The
second number is a Docker cache-hit rebuild.

## Docker Image Build Measurement

Run the stable compiler image build and keep the log:

```sh
cd /root/rust-anneal-playground/compiler
CHANNELS_TO_BUILD=stable bash ./build.sh 2>&1 | tee /tmp/anneal-docker-build-current.log
```

Print only the duration number, in milliseconds:

```sh
grep '\[anneal-build-timing\].*image=rust-stable.*event=finish' \
  /tmp/anneal-docker-build-current.log \
  | tail -n 1 \
  | sed -E 's/.*elapsed_ms=([0-9]+).*/\1/'
```

## First/Cold Anneal Verification Measurement

Restart the backend first so the request is cold for this backend process. Then
run one Anneal Verify request in the browser against the tiny test workload.

Current cold-run measurement from the VM:

```text
cold_backend_total_ms: 32310
cold_cargo_anneal_verify_ms: 31767
cold_backend_overhead_ms: 543
cold_success: true
```

Use a dedicated backend log:

```sh
/tmp/playground-ui-cold.log
```

Print exactly two duration numbers from the first cold run, in milliseconds:

```sh
grep '\[anneal-verify-timing\].*event=finish' /tmp/playground-ui-cold.log \
  | tail -n 1 \
  | sed -E 's/.*backend_total_ms=([0-9]+).*cargo_anneal_verify_ms=([0-9]+).*/\1 \2/' \
  | tr ' ' '\n'
```

Interpretation:

```text
line 1: cold_backend_total_ms
line 2: cold_cargo_anneal_verify_ms
```

If the timing line says `cargo_anneal_verify_ms_found=false`, the backend did
run Anneal but did not see the inner command timing marker in stdout/stderr.
Rebuild the backend from a version where Anneal Verify is wrapped with the
`[anneal] verification succeeded in ... ms` marker.

Then calculate:

```text
cold_backend_overhead_ms =
  cold_backend_total_ms - cold_cargo_anneal_verify_ms
```

## Warm Repeated Anneal Verification Measurement

Warm repeated measurements should reuse one live WebSocket session so the first
request can absorb session/container setup effects and the following requests
show repeated verifier cost. The public WebSocket client can confirm real
verifier execution by requiring the inner `[anneal] verification ... ms` marker,
but exact backend totals still come from `/tmp/playground-ui-cold.log`.

Current public WebSocket sequence from 2026-07-24:

```text
run  success  client_total_ms  cargo_anneal_verify_ms  client_observed_overhead_ms  marker_found
1    true     32352            31494                   858                          true
2    true     13078            12478                   600                          true
3    true     13019            12232                   787                          true
4    true     13017            12659                   358                          true
5    true     14022            13192                   830                          true
```

Use run 1 as the session setup/cold-in-session point. Runs 2-5 are the current
warm repeated sample:

```text
warm_repeated_cargo_anneal_verify_ms: 12478, 12232, 12659, 13192
warm_repeated_cargo_anneal_verify_avg_ms: 12640
warm_repeated_client_total_avg_ms: 13284
warm_repeated_client_observed_overhead_avg_ms: 644
```

Because these totals were collected over the public WebSocket instead of by
reading the VM log directly, treat `client_total_ms` and
`client_observed_overhead_ms` as close proxies for backend totals, not the
authoritative `backend_total_ms` and `backend_overhead_ms` fields.

To backfill exact backend fields for the same run window, SSH to the VM and run:

```sh
grep '\[anneal-verify-timing\].*event=finish' /tmp/playground-ui-cold.log | tail -n 6
```

## Inner Anneal Verify Phase Notes

Existing `RUST_LOG=ui=info,orchestrator=info,cargo_anneal=trace` output already
shows several useful inner phases. In the five-run WebSocket sequence above:

```text
phase                         run 1              warm runs 2-5
Charon                        250.57 ms          avg 132.57 ms
Aeneas                        361.80 ms          avg 369.45 ms
Lake build                    22.72 s            avg 4.22 s
Unattributed / diagnostics    about 8.16 s       avg about 7.92 s
```

The biggest observed cold-in-session delta is Lake build: it drops from about
22.7s on run 1 to about 4.0-4.4s on warm repeats. Charon and Aeneas are not the
dominant costs for the tiny workload. The remaining warm cost is mostly outside
the currently timed Charon/Aeneas/Lake lines, so the next useful instrumentation
target is to split materialization, Lake build, and Lean diagnostics with
explicit timing inside patched `cargo-anneal`.

## Stable Anneal Prewarm Experiment

Set `PLAYGROUND_ANNEAL_PREWARM_STABLE=1` when starting the backend to run the
tiny Anneal workload once after the backend successfully binds its port. The
warmup keeps the warmed stable coordinator alive and hands it to the first
WebSocket session that connects, so the first user-visible Verify with Anneal
can test whether it gets the warm Lake path.

The startup command should include:

```sh
PLAYGROUND_ANNEAL_PREWARM_STABLE=1
```

Useful log markers:

```text
[anneal-prewarm] starting stable Anneal prewarm
[anneal-prewarm] stable Anneal prewarm finished
[anneal-prewarm] using prewarmed stable coordinator for WebSocket
```

If the browser connects before the startup prewarm finishes, the backend logs:

```text
[anneal-prewarm] no prewarmed coordinator ready for WebSocket
```

For the experiment, wait until the `stable Anneal prewarm finished` line appears,
open a fresh browser tab, then run Verify with Anneal once against the tiny test
workload. Compare that first user-visible run's Lake build time against the
previous cold-ish Lake time (~20-21s) and warm Lake time (~3.8-4.2s).

## Persistent Anneal Target Experiment

Set `PLAYGROUND_ANNEAL_PERSISTENT_TARGET=1` when starting the backend to mount a
Docker named volume at `/playground/anneal-workspace/target/anneal` in each
compiler container. The volume name is channel-specific:

```text
playground-anneal-target-stable
playground-anneal-target-beta
playground-anneal-target-nightly
```

The first run that creates an empty Docker named volume should initialize it
from the compiler image, so the baked `cargo_target` cache remains available.
Subsequent compiler containers can then reuse the runtime-generated Lean
workspace and `.lake` build output across backend and container restarts.

Use it together with startup prewarm to test whether the hidden prewarm itself
becomes warm after a restart:

```sh
PLAYGROUND_ANNEAL_PERSISTENT_TARGET=1
PLAYGROUND_ANNEAL_PREWARM_STABLE=1
```

Useful log marker:

```text
[anneal-cache] mounting persistent Anneal target volume
```

Expected measurement:

1. Start once with the persistent target enabled and let the stable prewarm
   finish. The first prewarm may still pay the cold Lake cost if the volume was
   empty.
2. Restart the backend with the same env vars.
3. The next startup prewarm should show a warm `lake build` time, roughly in the
   same 4s band as the previous warm user-visible runs.

To reset the stable-channel experiment:

```sh
docker volume rm playground-anneal-target-stable
```

## Test Workload

```rust
/// ```anneal, unsafe(axiom)
/// ```
pub unsafe fn anneal_warmup_identity(x: u32) -> u32 {
    x
}

fn main() {}
```
