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

Then calculate:

```text
cold_backend_overhead_ms =
  cold_backend_total_ms - cold_cargo_anneal_verify_ms
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

