---
description: Analyze an Android ANR (Application Not Responding) — read the trace, locate the main-thread blocking point, classify the root cause, and recommend a minimal off-main-thread fix. Invokes the android-anr-analyzer agent.
argument-hint: "[path/to/traces.txt | bugreport.zip | pasted stack | package name]"
---

# Analyze Android ANR

This command invokes the **android-anr-analyzer** agent to diagnose an ANR with read-only,
evidence-based analysis.

## What This Command Does

1. **Locate the trace** — read a provided `traces.txt`/bugreport/inline stack, or pull from a connected device
2. **Identify the ANR type** — input dispatch (5s), broadcast (10/60s), service (20/200s), provider (10s)
3. **Find the blocking frame** — the first app frame stuck on the `main` thread (`tid=1`)
4. **Classify the root cause** — blocking I/O, lock/deadlock, sync Binder/IPC, heavy CPU work, queue congestion, lifecycle overrun, or system-level
5. **Recommend a minimal fix** — move work to the correct Coroutine dispatcher and the correct layer

## When to Use

Use `/anr` when:
- An app shows the "isn't responding" dialog or freezes
- You have an ANR `traces.txt`, an `anr_<timestamp>` file, or a bugreport to analyze
- logcat shows `ANR in`, `Slow operation/dispatch`, or `Skipped N frames`
- You want a pre-ship review for main-thread I/O, lock contention, or synchronous IPC

## Usage

```text
/anr ./anr-traces.txt
/anr bugreport-2026-06-20.zip
/anr com.example.app          # pulls from a connected device via adb
/anr <paste the main-thread stack inline>
```

## Notes

- This is a **read-only** diagnostic. No app code, build config, or device state is changed
  unless you explicitly ask for a fix to be applied.
- For the underlying Android concurrency and dispatcher guidance, see the
  `android-anr-analysis` and `kotlin-coroutines-flows` skills.
