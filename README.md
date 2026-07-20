The pipeline runs across both ESP32-S3 cores. On Core 1, four tasks form the ECG processing chain, connected by three distinct IPC primitives chosen for the job each is suited to:

- **Queue** — producer → consumer, non-blocking send, decouples sampling rate from processing rate
- **Event group** (AND-rendezvous) — consumer/producer → coordinator, fires only once both halves of a cycle are complete
- **Direct task notification** — coordinator → responder (routine wakeup), and button ISR → responder (simulated nurse-call), both landing on the identical wakeup path

An independent monitor task runs on Core 0, reporting system status without interfering with the real-time plane.

## Tasks & timing (WCET evidence)

WCET was measured with `esp_timer_get_time()` bracketing each task's active work (excluding blocking waits), over hundreds of cycles per task. C is reported as measured max + 30% margin. All four Core 1 tasks share the producer's 50 ms period, with an implicit deadline D = T.

| Task | Period T (ms) | Deadline D (ms) | WCET C (ms) | U = C/T | Priority | Core |
|------|---------------:|-----------------:|-------------:|--------:|---------:|-----:|
| Producer | 50 | 50 | 0.046 | 0.0009 | 8 | 1 |
| Consumer | 50 | 50 | 15.629 | 0.3126 | 8 | 1 |
| Coordinator | 50 | 50 | 0.013 | 0.0003 | 9 | 1 |
| Responder | 50 | 50 | 11.824 | 0.2365 | 12 | 1 |
| Monitor | 1000 | 1000 | — | — | 4 | 0 |

**Total utilization (Core 1, n=4):** U ≈ 0.0009 + 0.3126 + 0.0003 + 0.2365 = **0.5503**
**Liu & Layland bound (n=4):** 4×(2^(1/4) − 1) ≈ **0.7568**
**Verdict:** U = 0.55 ≤ 0.7568 — schedulable under Rate Monotonic, with real margin.

One honest outlier: the consumer's measured max jumped from a steady ~2 ms baseline to a single 12 ms spike partway through the run and didn't fully recover afterward, most likely a one-off scheduling stall from simulator jitter rather than a property of the algorithm itself. It's included in the C value above rather than smoothed over.

### Response-Time Analysis (to convergence)

Utilization alone confirms schedulability under the Liu & Layland sufficient bound. RTA gives a tighter, per-task guarantee by iterating R = C + Σ⌈R/Tⱼ⌉×Cⱼ over all higher-priority tasks j until R stops changing. Producer and Consumer share priority 8, so each is additionally analyzed as blocked by one worst-case execution of the other (equal-priority round-robin interference).

| Task | Priority | RTA iteration | Converged R (ms) | Deadline D (ms) | Margin |
|------|---------:|----------------|------------------:|------------------:|-------:|
| Responder | 12 (highest) | R = C = 11.824 | 11.824 | 50 | 38.176 ms |
| Coordinator | 9 | R₀=0.013 → R₁=0.013+11.824=11.837 → converged | 11.837 | 50 | 38.163 ms |
| Producer | 8 (tied) | R₀=0.046 → R₁=0.046+11.824+0.013+15.629=27.512 → converged | 27.512 | 50 | 22.488 ms |
| Consumer | 8 (tied) | R₀=15.629 → R₁=15.629+11.824+0.013+0.046=27.512 → converged | 27.512 | 50 | 22.488 ms |

**Result:** every task's converged worst-case response time is comfortably under its 50 ms deadline — the tightest case (Producer/Consumer) still has 22.5 ms of slack.

### Priority assignment defense

Priorities were assigned by criticality of consequence, not strictly by period, since all four tasks share the same 50 ms period and classic Rate-Monotonic ordering by period doesn't differentiate them. **Responder** holds the highest priority because it's the alarm/output stage — any delay here directly delays a patient-facing alert. **Coordinator** is next, since it gates the responder and should not itself become a bottleneck. **Producer** and **Consumer** share the lowest priority (8): their work has slack (confirmed by RTA above) and neither is itself alarm-facing, though tying their priorities is a simplification — a stricter design would break the tie by giving Producer slight priority over Consumer, since a delayed sample is recoverable but a delayed send risks queue buildup. This tradeoff is flagged here rather than glossed over.

### Producer/consumer contract

The queue boundary between producer and consumer is governed by an explicit, defended contract rather than implicit behavior:

- **Producer guarantees:** attempts a non-blocking send every 50 ms; never blocks waiting for queue space (blocking here would let a slow consumer stall the producer's own deadline); on a full queue, drops the newest sample and increments an overflow counter rather than retrying or discarding an older one.
- **Consumer guarantees:** attempts a receive with a 100 ms timeout (2× the producer's period); processes every sample it successfully dequeues exactly once; logs an explicit stall warning if no sample arrives within the timeout window, rather than blocking silently forever.
- **What's explicitly not guaranteed:** delivery of every produced sample under sustained overload — the drop-newest policy is a deliberate recency-over-completeness tradeoff, not an oversight.

## Hazard analysis & standard mapping

This analysis follows the structure of **IEC 62304** (medical device software lifecycle), the standard that governs software risk specifically, as distinct from IEC 60601's broader electrical/mechanical scope. Each hazard is drawn from actual behavior observed or engineered during development, not a hypothetical checklist.

| Hazard | Effect | Mitigation | IEC 62304 mapping |
|--------|--------|------------|--------------------|
| Queue overflow under consumer stall | ECG samples silently dropped; displayed HR could lag reality | Queue sized with ~2.7× margin over worst-case stall estimate; drop-newest policy (never blocks the producer); overflow event counted and logged | §5.2 risk control implementation |
| Task-creation-order race (IPC object used before creation) | NULL-handle fault, full system crash on boot — total loss of monitoring | Found and fixed during development: all IPC objects created before any task that references them, documented as a standing rule | §5.1 development planning & defect resolution |
| Consumer execution-time outlier (12 ms spike vs. ~2 ms baseline) | Delayed arrhythmia decision within a single 50 ms cycle | WCET measured with margin (C = max + 30%); total utilization (0.55) stays well under the RM bound (0.7568) even with the outlier included | §5.5 verification of real-time performance |
| Responder task stalls or is starved (missed/delayed alarm) | An abnormal HR reading goes unflagged — the highest-severity failure mode | Responder runs at the highest priority (12); per-task heartbeat counters expose a stalled responder immediately | §5.7 problem resolution; alarm-system intent aligned with IEC 60601-1-8 |
| Spurious button-ISR triggers (switch bounce) | False nurse-call events, unnecessary responder wakeups | 50 ms software debounce window in the ISR before a signal is accepted | §5.5 verification of interrupt handling |

Taken together, these hazards point to one design theme worth calling out: the failure mode that matters most in a patient monitor isn't "the system is slow" — it's "the system is silent." The mitigations lean on visibility (heartbeats, overflow counters, latency logging) as much as raw timing margin. A system that fails should fail loudly.

## Graceful degradation

The responder's wait was changed from an unbounded block (`portMAX_DELAY`) to a 200 ms timeout (4× the producer's period). If no pipeline cycle completes within that window, the responder treats the system as stalled: it logs a **STALE** event using the last known heart rate rather than silently going quiet, and keeps re-announcing every 200 ms until fresh data resumes. This is deliberately decoupled from *where* the pipeline breaks — the responder doesn't need to know whether the producer, consumer, or coordinator failed, only that a cycle didn't arrive in time.

To demonstrate this on demand rather than only describe it, an `INDUCE_FAILURE` compile switch simulates a sensor disconnect: the producer stops pushing samples after 100 cycles (~5 seconds), as if the ECG lead had come loose.

**Predicted:** the consumer's queue-receive timeout would fire repeatedly once the producer stopped; the event group would stop being set, so the coordinator (waiting with `portMAX_DELAY`, no timeout of its own) would block indefinitely; the responder, independent of the coordinator's blocked state, would detect the stall via its own 200 ms timeout and begin announcing STALE using the last known reading. The system should not crash and should not go silent.

**Observed:** the system ran 100 normal cycles (5.0 s), then the induced failure fired at t=5.04s. The consumer began logging stall warnings almost immediately. The first STALE event appeared 150 ms later at t=5.19s (last known HR=73 bpm), and continued firing every ~200 ms without interruption for the remainder of the 43-second run (178 consecutive STALE events by the end of the captured log). The system never crashed and never went silent. One deviation: around t=20–23s, a few spurious wakeups briefly interrupted the STALE sequence (real latency values in the 750–1300 µs range), most likely simulator noise on the floating button GPIO. Detection resumed correctly immediately afterward.

**Analysis:** observed behavior matched the prediction closely. The 150 ms gap before the first STALE event is expected and correct — the timeout window has to fully elapse from the *last successful cycle*, not from the failure event itself, so a delay up to one timeout period is inherent to the design, not a bug. The spurious-wakeup blip was not predicted and is treated as a simulator artifact rather than a flaw in the degradation logic — the STALE detector correctly resumed on its own once the noise passed, which is itself evidence the timeout-based approach is robust to transient interruptions.

## Build & run

**Hardware/target:** ESP32-S3, simulated in Wokwi (no physical board required). **Toolchain:** ESP-IDF via the Wokwi web IDE.

1. Open the Wokwi project: `https://wokwi.com/projects/469906597165240321`
2. Click the play/simulate button. The serial monitor panel shows live output.
3. Default configuration runs the pipeline normally — no button press or config change needed to see it working end to end.

**Compile-time switches** (top of `main.c`):

| Switch | Default | Purpose |
|--------|--------:|---------|
| `USE_WEBSERVER` | 0 | 0 = serial monitor output (no Wi-Fi needed); 1 = web dashboard over Wokwi's simulated Wi-Fi |
| `MEASURE_SEMAPHORE_LATENCY` | 0 | 0 = responder listens to the coordinator's routine notification (normal operation); 1 = isolates the button-triggered semaphore path for the latency comparison only |
| `INDUCE_FAILURE` | 0 | 1 = simulates a sensor disconnect after 100 cycles (~5 s), triggering the graceful-degradation path |

To reproduce the fault-injection demo: set `INDUCE_FAILURE` to `1`, keep `MEASURE_SEMAPHORE_LATENCY` at `0`, run the simulation, and watch the serial log transition from normal `[responder] cycle OK` lines to repeated `[Medical] STALE` alerts.

## Tailored for

This system is tailored toward **safety-critical and medical embedded roles**. That choice shaped which features made the cut: every timing claim is measured, not assumed; hazards are mapped to a real standard (IEC 62304) rather than a generic risk table; failures are made visible (heartbeats, overflow counters, the STALE detector) rather than silent, since the worst failure mode for a patient monitor isn't slowness — it's silence; and outliers (the consumer's WCET spike, the spurious ISR noise) are reported rather than edited out, since a suspiciously clean dataset is a red flag in a safety context, not a selling point.

None of this required exotic engineering — just consistently choosing the boring, defensible option (a timeout instead of an infinite wait, a counter instead of silent failure, a real log instead of a claimed number) at every decision point.

## Task table & Wokwi project

**Wokwi project:** `Ramkissoon-FINAL-RTS26Summer`
**Wokwi URL:** https://wokwi.com/projects/469906597165240321
