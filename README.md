# Project Aeon-Trace — The Continuous Continuum Recalibrator

Deterministic, closed-form recalibration of fundamental physical constants
(`α`, `G`, `c`) from continuous scalar-field differentials. No probabilistic
models, no data fitting, no machine learning — every output is a pure
function of its inputs. Standard library only, zero third-party
dependencies — including the live dashboard's web server.

## Build, run, and test

```powershell
cargo build          # compile the engine
cargo run             # launch the live web dashboard for this computer and your LAN
cargo run -- cli      # print a one-shot text report (sample telemetry)
cargo run -- 0.01 0.08 1.0 1.0 1.0025 0.0025   # one-shot report, custom telemetry
cargo test            # run the full unit test suite (22 tests)
```

The dashboard listens on `0.0.0.0:7878` by default, so other devices on the
same Wi-Fi/LAN can open it while the app is running. The startup banner prints
the exact URL to use from another device. Override the bind address or port
with `AEON_HOST` and `AEON_PORT`; use `AEON_HOST=127.0.0.1` for local-only mode.

## Live dashboard

`cargo run` starts a zero-dependency HTTP server (hand-rolled on
`std::net`/`std::io` — no web framework) bound to all local network interfaces
by default. It serves a dark, sci-fi styled single-page dashboard
(`assets/index.html`) where you can edit telemetry, hit
**RECALIBRATE**, **SAMPLE**, or **RANDOMIZE**, and watch the gradients,
invariant strain, and recalibrated `α`/`G`/`c` update live, backed by the
same engine and validation as the CLI. The JSON API it calls is
`GET /api/recalibrate?phi_x1=..&phi_x2=..&delta_x=..&phi_t1=..&phi_t2=..&delta_t=..`.

## Data transformation pipeline

```text
┌───────────────────────┐     ┌───────────────────────┐
│ Telemetry Inputs      │     │ Telemetry Inputs      │
│ Φ(x1), Φ(x2), Δx      │     │ Φ(t1), Φ(t2), Δt      │
└──────────┬────────────┘     └──────────┬────────────┘
           │                             │
           ▼                             ▼
┌───────────────────────┐     ┌───────────────────────┐
│ Spatial Gradient      │     │ Temporal Gradient     │
│ ∇xΦ = (Φx2-Φx1)/Δx    │     │ ∇tΦ = (Φt2-Φt1)/Δt    │
└──────────┬────────────┘     └──────────┬────────────┘
           │                             │
           └──────────────┬──────────────┘
                           ▼
              ┌─────────────────────────┐
              │ Invariant Metric Strain │
              │ S = (∇tΦ)² - (∇xΦ)²     │
              └────────────┬────────────┘
                            ▼
              ┌─────────────────────────┐
              │ Euler Metric Scaling    │
              │ K_new = K0 · e^(ξK · S) │
              │ (applied per constant:  │
              │  α, G, c)               │
              └────────────┬────────────┘
                            ▼
              ┌─────────────────────────┐
              │ Audit Report            │
              │ raw values + % drift    │
              └─────────────────────────┘
```

## Plain-language explanation

1. Two sensors report a scalar field reading `Φ` at two nearby points in
   space, and two readings at two nearby moments in time.
2. Dividing each reading's *change* by the *distance/time it changed over*
   gives a rate of change — a "gradient" — one for space, one for time.
3. Special relativity treats space and time asymmetrically (the `(+,-,-,-)`
   signature), so the two gradients are combined as `time² - space²` into a
   single number called the "invariant strain" — how much the local fabric
   of spacetime is distorted.
4. Each physical constant has its own sensitivity ("coupling factor") to
   that distortion. The constant is rescaled by `e^(coupling × strain)`, an
   exponential curve that always returns exactly the original value when
   strain is zero, and smoothly grows or shrinks the constant when strain is
   non-zero.
5. The engine reports the new value of each constant alongside the
   percentage it drifted from its known baseline, so a human can sanity
   check the magnitude of the shift at a glance.

## Robustness

Every input is validated before use:

- Differential steps (`Δx`, `Δt`) must be non-zero, otherwise the engine
  returns `RecalibrationError::ZeroDifferential` instead of dividing by zero.
- Field readings must be finite, otherwise the engine returns
  `RecalibrationError::NonFiniteInput`.
- If an extreme strain saturates a constant to a non-finite value, the
  report status becomes `SATURATION_WARNING` instead of silently reporting
  `DETERMINISTIC_LOCK_VERIFIED`.

## Test coverage (`aeon_trace`, `#[cfg(test)] mod tests`)

- `scale_constant_at_zero_strain_returns_baseline_exactly` — zero strain ⇒
  `K_new == K0`.
- `gradient_calculation_matches_known_values` — spatial/temporal gradient
  arithmetic against hand-computed values.
- `zero_metric_strain_leaves_constants_unperturbed` — full-pipeline zero
  strain edge case.
- `high_spatial_gradient_drives_negative_drift` — dominant spatial gradient
  edge case.
- `temporal_oscillation_produces_finite_alternating_strain` — oscillating
  field readings stay finite and produce both rising and falling gradients.
- `zero_delta_x_is_rejected` / `zero_delta_t_is_rejected` — division-by-zero
  guard.
- `non_finite_input_is_rejected` — NaN/inf input guard.
- `run_audit_renders_report_without_panicking` — end-to-end audit report
  rendering.
- `parse_telemetry_accepts_six_valid_numbers` / `_rejects_non_numeric_input`
  — CLI argument parsing.

## Test coverage (`server`, `#[cfg(test)] mod tests`)

- `parse_request_line_*` — HTTP request-line parsing, including malformed input.
- `percent_decode_*` — query-string decoding, including truncated `%` escapes.
- `parse_query_builds_expected_map` — query-string-to-map parsing.
- `json_number_uses_sentinels_for_non_finite_values` /
  `json_escape_escapes_quotes_and_backslashes` — safe JSON encoding.
- `handle_recalibrate_returns_error_json_for_missing_param` /
  `_returns_success_json_for_valid_query` — API handler correctness.
- `dashboard_serves_index_and_api_over_real_tcp` — end-to-end integration
  test over a real TCP connection to an ephemeral port.
