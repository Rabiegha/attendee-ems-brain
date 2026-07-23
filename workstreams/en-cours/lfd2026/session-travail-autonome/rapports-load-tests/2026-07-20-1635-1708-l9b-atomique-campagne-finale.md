# LFD k6 Campaign — Atomic L9.b and Measured Limit

## Executive Verdict

- Environment: **staging only**, with production monitored in read-only mode.
- Application fix: backend PR #50, reserving the seat and creating the choice in a single atomic
  PostgreSQL statement.
- Main SHA tested: `4bd938d55f11576d1a25291952fe725f636d535c`, followed by
  `f37df976dcbb18b1e8a67d5de92e9ed990ff8f71` after the disabled Gotenberg B0 client was added,
  without being connected to the registration flow.
- **Proven sustained throughput: 70 registrations/s for two minutes.**
- **Proven simultaneous burst: 250 registrations**, started within a 10 ms window.
- Sustained 75/s, 350 simultaneous registrations, and 500 simultaneous registrations have not been
  validated.
- The contractual requirement of 3,000 simultaneous registrations **was not proven** by this
  single-generator campaign and must not be extrapolated from the 250-registration result.

## Sustained-Load Results

| Stage | Effective duration | Confirmations | Throughput | p95 | p99 | Max | Errors/losses | Verdict |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| smoke | 1 request | 1/1 | — | 157 ms | 157 ms | 157 ms | 0 | green |
| 25/s | 120 s | 3,001/3,001 | 24.99/s | 113.96 ms | 156.45 ms | 296.51 ms | 0 | green |
| 50/s | 120 s | 6,001/6,001 | 49.99/s | 164.68 ms | 237.76 ms | 392.79 ms | 0 | green |
| 60/s | 120 s | 7,201/7,201 | 59.97/s | 149.62 ms | 247.70 ms | 438.57 ms | 0 | green |
| 65/s | 120 s | 7,801/7,801 | 64.98/s | 205.88 ms | 530.91 ms | 1.84 s | 0 | green after manual reconciliation |
| 70/s | 120 s | 8,400/8,400 | 69.97/s | 222.98 ms | 405.37 ms | 1.00 s | 0 | green |
| 75/s | stopped at ~4 s | 107 received | 26.89/s | 2.39 s | 2.67 s | 3.10 s | 49 dropped | red |

The 75/s stage created 248 registrations in the database after the test was stopped, while only 107
confirmations were received. This illustrates the UX risk when a client is interrupted while
transactions already in progress eventually commit. The counters remained accurate and capacity
was never exceeded.

## Simultaneous-Burst Results

| Burst | Start spread | Confirmed responses | p95 | p99 | Max | DB result after cooldown | Verdict |
| --- | ---: | ---: | ---: | ---: | ---: | --- | --- |
| 100 | 4 ms | 100/100 | 2.15 s | 2.28 s | 2.37 s | exactly 100 | green |
| 250 | 10 ms | 250/250 | 497.73 ms | 637.49 ms | 792.34 ms | exactly 250 | green |
| 350 | 11 ms | 321 received, 2 timeouts | 1.02 s* | 1.36 s* | 1.64 s* | 327, including 6 with no confirmation received | red |
| 500 | 15 ms | 448 received, 12 timeouts | 3.90 s* | 5.08 s* | 5.54 s* | exactly 448 | red |

`*` Percentiles from interrupted runs cover only the responses observed before the test was stopped.
They do not represent users who were still blocked or interrupted and must not be interpreted as a
successful UX result.

The fact that the 250-registration burst had better latency than the 100-registration burst also
shows the variability of a single sample due to caching, checkpoints, and scheduling. The robust
conclusion is complete success or failure with invariants, rather than a fine-grained comparison
between these two isolated percentile results.

## Business Invariants and Audit

- Every completed run kept `Session.registered_count` equal to the actual number of confirmed
  choices and never exceeded capacity.
- Green runs had exactly one attendee, one email, one registration, one confirmed choice, and one
  HTTP 2xx audit entry per successful operation.
- The 350-registration run revealed six commits for which no response was received: 327
  registrations/choices in the database versus 321 observed 2xx audit entries. This confirms that
  idempotency and UX reconciliation remain essential beyond the safe operating zone.
- No restart, OOM event, or inconsistent final state was attributable to the load. A separate 65/s
  run was deliberately invalidated when the watchdog detected the concurrent B0 deployment; the
  three 502 responses in that run do not represent a capacity limit.

## Observed Resources

| Stage | API peak CPU | PostgreSQL peak CPU | Minimum available host memory |
| --- | ---: | ---: | ---: |
| 25/s | 176.47% | 49.74% | 20,083 MB |
| 50/s | 266.23% | 81.09% | 20,004 MB |
| 60/s | 275.88% | 100.90% | 19,740 MB |
| 70/s | 363.88% | 155.63% | 19,734 MB |

Docker CPU percentages can exceed 100% when multiple cores are used. The host has eight cores. The
load generator remained well below its memory and CPU thresholds and was not the limiting factor
for these stages. The health of the shared production environment remained green during read-only
checks.

## Evidence Incident and Test-Harness Fix

After the 65/s run, the final reconciliation query exceeded its 30-second timeout. It filtered
`audit_logs` by `target_id` without specifying `target_type`, while the existing index is the
composite index `(target_type, target_id)`. Backend PR #52 adds
`target_type = 'session_registration'`: the same verification then completed immediately and
confirmed exactly 7,801/7,801 items. The k6 run is valid; the initial failure affected only the
transport of the post-flight evidence.

## Recommendation for LFD

- **Measured technical GO** for 70 sustained registrations/s for two minutes on this instance and
  for a burst of 250 genuinely near-simultaneous registrations.
- **NO-GO for claiming sustained 75/s, 350 simultaneous, 500 simultaneous, or 3,000 simultaneous
  registrations.**
- 70/s represents approximately 4,200 registrations per minute if demand is evenly distributed;
  this figure does not replace a burst test.
- Proving 3,000 requires multiple synchronized generators, a shared barrier, measurement of the
  start spread, and global reconciliation. The current harness deliberately rejects more than 500
  VUs on one generator and rejects multi-generator runs until the orchestrator is delivered.
- At a minimum, rerun sustained 70/s and 250 simultaneous registrations after any change that
  actually affects the registration path, synchronous audits, triggers, or the database pool. The
  isolated Gotenberg B0 client does not affect this path.

## Traceability

- PR #50: `perf(registration): atomically claim session choice`.
- PR #51: Gotenberg OIDC B0 client, isolated and disabled by default.
- PR #52: use of the audit index in k6 reconciliation.
- Local raw artifacts:
  `attendee-ems-back-wt-k6-final/tmp/k6/lfd-l9b-{atomic,b0}-*/gen-01/`.
- Each directory contains metadata, load configuration, the k6 summary, resource measurements,
  watchdog data, live probes, and the reconciliation result. No token or credential is included in
  this report.
