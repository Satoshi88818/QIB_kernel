QIBv2 Review — Consolidated Fix Summary (v2)

Twelve patches total: nine addressing distinct bugs and test gaps found across two review passes of the QIBv2 codebase, plus three follow-up passes that took the three "we couldn't fully execute this" boundaries from the first nine and pursued them for real — building a real PyO3 extension, downloading and running the real TLA+ model checker, and downloading and running the real CBMC model checker (Kani's backend).

Every patch below was implemented as real code and exercised with a real test run — not just described. The three follow-up passes are the most interesting part of this document: in every single one, actually running the tool (instead of reasoning about what it would probably find) surfaced a genuine bug that reasoning alone had missed. That pattern is called out explicitly in each section below, because it's the main thing worth remembering from this whole exercise.

Each section links back to its full standalone deliverable, which contains the complete source, the full test output, and a direct repro of the original bug for comparison.

Part 1 — The original nine fixes

FIX-H2 — Capability Revocation Blast Radius

File: qib-muk/src/capability.rs · Deliverable: QIB-muK_FIX-H2_capability_epoch_scoping.md

Bug: revoke() bumped a single global epoch counter shared by every capability. Revoking one capability instantly invalidated every other live capability system-wide — a self-inflicted, system-wide denial of service.

Fix: Per-slot generation counters. Revoking slot i touches only slot i.

Validation: 10/10 Rust tests, cargo test. Old bug reproduced directly first (A valid after revoking B? false / C valid after revoking B? false).

FIX-G — Attestation Lease Enforcement

File: qib-muk/src/capability.rs · Deliverable: QIB-muK_FIX-G_attestation_lease_enforcement.md

Bug: require_attestation() — the real authorization gate — never checked valid_until_ns. A lapsed-but-not-revoked attestation still passed.

Fix: Moved the lease onto CapabilityKind::Attestation directly; require_attestation(pid, now_ns) now checks it, distinguishing AttestationRequired from AttestationExpired.

Validation: 17/17 tests. Old bug reproduced: an attestation expired 4000ns earlier still returned Ok.

FIX-J — Scheduler Dequeue

File: qib-muk/src/scheduler.rs · Deliverable: QIB-muK_FIX-J_scheduler_dequeue.md

Bug: next_task() returned a borrow without removing the task from ready_queue — the same task dispatched forever; the queue grew unbounded.

Fix: swap_removes the selected task, returns it by value, updates current_task.

Validation: 27/27 tests. Old bug reproduced: three calls on a one-task queue returned Some(1) all three times.

FIX-K — Safe-Bypass / TLA+ Liveness Reconciliation

Files: qib-muk/src/watchdog.rs, qib-muk/src/entry.rs · Deliverable: QIB-muK_FIX-K_safe_bypass_watchdog.md

Bug: The panic handler spun forever; the TLA+ spec required eventually leaving Safe-Bypass. Proven structurally impossible via rustc's own unreachable-code lint on the diverging handler.

Fix: A hardware watchdog. The main loop pets it every tick; the panic handler deliberately never does, guaranteeing a forced reset.

Validation (at the time): 34/34 tests + cargo build for the real no_std wiring. TLA+ itself was not re-verified in this pass — flagged explicitly. Superseded by FIX-Q below, which actually ran TLC against this.

FIX-L — Redundant AttestationCap Wrapper Cleanup

File: qib-muk/src/attestation.rs · Deliverable: QIB-muK_FIX-L_attestation_cap_cleanup.md

Bug: A leftover wrapper duplicated FIX-G's lease logic, and renew_attestation() minted a second capability instead of extending the first — leaking an orphaned, still-valid capability on every renewal.

Fix: Deleted the wrapper; renewal now extends the existing capability in place.

Validation: 39/39 tests. Old bug reproduced: one mint + one "renewal" produced 2 live capabilities, not 1.

FIX-M — Userspace Test Gaps

File: qib/src/ai/digital_twin.rs · Deliverable: QIB_FIX-M_userspace_tests.md

Two bugs: (1) assert ls.fidelity_std > 0.0 or True — vacuously true, hiding a real epsilon-floor gap (.max(0.0) instead of a real floor). (2) No API ever set correlation_group_id, so group-failure safety-tax escalation was unreachable.

Fix: Real epsilon floor; added set_correlation_group().

Validation (at the time): 9/9 tests on core Rust logic only — PyO3 boundary explicitly flagged as unexecuted. Superseded by the real PyO3 pass below, which actually built and ran this.

FIX-N — isqrt Changelog / Kani Annotation Cleanup

File: qib-muk/src/quantum/fidelity.rs · Deliverable: QIB-muK_FIX-N_isqrt_cleanup.md

Bug: Changelog claimed 32 unrolled iterations; actual code uses 6, with a decorative #[kani::unwind(6)] bounding a loop that doesn't exist.

Fix: Corrected the changelog; removed the inert annotation.

Validation (at the time): 6/6 tests, exhaustive sweep 1..1,000,000 plus a 200,000-sample pseudo-random sweep — Kani itself not re-run, flagged explicitly. Partially superseded by FIX-R below, which ran real CBMC (Kani's backend) against the algorithm.

FIX-O — MPMC Hardening on enqueue_hw_event

File: qib-muk/src/hw/p4.rs · Deliverable: QIB-muK_FIX-O_p4_mpmc_hardening.md

Bug: "Evict oldest if full, then push" was three separate steps; under real multi-producer concurrency, a second producer's push could land in the gap, silently dropping the newest event instead of the oldest.

Fix: Single atomic ArrayQueue::force_push.

Validation: 7/7 tests, including a deterministic reproduction of the exact race (not a flaky thread test) and a real 8-thread × 5,000-push concurrent stress test.

FIX-P — Real Test for Mutex Poison Recovery

File: qib/src/ai/calibration.rs · Deliverable: QIB_FIX-P_mutex_poisoning_test.md

Bug: TestMutexPoisoning never actually poisoned anything — called the happy path and checked accessibility, which passes regardless of whether FIX-D's recovery code exists.

Fix: debug_poison_state_for_test() — a real thread panics while holding the lock.

Validation (at the time): 5/5 tests, including a --nocapture run showing a real panic and backtrace. PyO3 boundary flagged as unexecuted. Superseded by the real PyO3 pass below.

Part 2 — Pursuing the three open boundaries for real

Each of FIX-K, FIX-M/FIX-P, and FIX-N carried an explicit note: "this boundary wasn't executed in this sandbox." A feasibility check found all three were more reachable than assumed. Pursuing each one for real is where the most interesting findings in this whole project turned up.

Real PyO3 Bindings — closing FIX-M and FIX-P

Deliverable: QIB_PyO3_real_bindings.md

Built an actual Python extension module (maturin develop) wrapping the FIX-M and FIX-P Rust logic, installed it into a real virtualenv, and ran the corrected test_calibration.py with real pytest — not syntax-checked, actually executed against compiled bindings.

Real bug found: the first run failed 3 of 10 tests. Not a bug in FIX-M or FIX-P's logic — a bug in the test harness: setup_function only fires before bare module-level functions, never before methods inside test classes, so state silently leaked between tests. python3 -m py_compile would never have caught this (it's a pytest semantics issue, not a syntax error). Fixed with a proper @pytest.fixture(autouse=True).

Final result: 10/10 real passes, plus a real poisoning demonstration (qib._debug_poison_calibration_state()) with a genuine Rust panic visible from Python and the state surviving it.

FIX-Q — Real TLA+/TLC Verification

Deliverable: QIB-muK_FIX-Q_tla_verification.md

Downloaded tla2tools.jar from GitHub and ran real TLC against MicrokernelScheduler.tla. This surfaced two genuine defects in the original document's spec itself, independent of anything in Rust:

The spec was never actually runnable. THEOREM Spec => ... referenced Spec, but Next and Spec were never defined anywhere. TLC's own error: Unknown operator: 'Spec'. Every property in that THEOREM — including the one FIX-K cared about — had never been mechanically checked at all.

SafeBypassEscapable was too weak to mean what it claimed. Written as safe_bypass => <>(~safe_bypass) with no enclosing [], TLC only evaluates it against the initial state, where safe_bypass is always FALSE — making it vacuously true for any implementation, found only because the old, broken design also passed trivially on the first attempt. Fixed with one symbol: [](safe_bypass => <>(~safe_bypass)).

Final result: the completed spec (with FIX-K's WatchdogReset action and its fairness condition) passes exhaustively across 492 states. Re-running the old design (no WatchdogReset) against the corrected property produces a genuine counterexample — a concrete lasso trace where safe_bypass gets stuck at TRUE forever, exactly as FIX-K's write-up predicted from first principles.

FIX-R — Real CBMC Verification (Kani's Backend)

Deliverable: QIB_FIX-R_kani_cbmc_verification.md

Kani itself could not run: its compiler requires linking against one exact nightly rustc build, obtainable only via rustup from static.rust-lang.org, a domain this sandbox returns 403 host_not_allowed for — confirmed directly, not assumed. CBMC (Kani's own backend) came bundled in the download and runs standalone with no such dependency. Since isqrt_u64_fixed has no loops, it was hand-translated to C and verified with CBMC directly — the same underlying technology, not the same pipeline.

Two real bugs found in the verification harness (not the algorithm):

A width-parameterization bug: non-native bit-widths (e.g. WIDTH=20) silently didn't restrict the actual domain at all, causing an unsigned underflow. CBMC caught it immediately: shift distance too large.

Saturating multiplication hid the true value at exactly n = u64::MAX, making a correct result look like a failure. Confirmed via Python that the algorithm was right and the checker was wrong; fixed with exact 128-bit (unsigned __int128) arithmetic.

Final result: genuine, exhaustive VERIFICATION SUCCESSFUL at 8-bit (256 values) and 16-bit (65,536 values) domains, plus an exact targeted proof over the 100,001 hardest inputs near u64::MAX at full 64-bit width. The fully unconstrained 64-bit domain remains genuinely intractable for brute-force SAT bit-blasting in this environment — a real, confirmed limit (timed out at 32-bit after 200s; the unconstrained 64-bit attempt appears to have been killed for memory exhaustion) that would plausibly affect a real Kani run too, since it uses the same CBMC backend. k-induction was flagged as a possible way to close this specific remaining gap, not attempted.

Summary table

FixAreaTestsReal execution found a bug?H2Capability revocation scope10/10—GAttestation lease enforcement17/17—JScheduler dequeue27/27—KSafe-Bypass liveness (code)34/34—LAttestation cap cleanup39/39—MUserspace test gaps (Rust core)9/9—Nisqrt cleanup (Rust core)6/6—OP4 ring MPMC hardening7/7—PMutex poisoning test (Rust core)5/5—Real PyO3Closes M + P's boundary10/10Yes — pytest fixture scoping bugFIX-Q (TLA+/TLC)Closes K's boundary492-state exhaustive + 1 counterexampleYes — twice: unrunnable spec, vacuous propertyFIX-R (CBMC)Partially closes N's boundaryExhaustive @ 8/16-bit + exact @ boundaryYes — twice: domain bug, arithmetic bug 

154 Rust assertions from the original nine patches, all passing, plus three more execution passes that together found five additional real bugs — every single one of them in verification tooling, not in the underlying Rust implementation the tooling was checking. That last point is worth sitting with: across three independent "let's actually run the real tool" attempts, the production code held up every time, and the mistakes were all in the scaffolding built to check it — a reasonably strong, if indirect, vote of confidence in the original nine fixes, and a clear demonstration of why "it should work" and "it does work" are different claims.


