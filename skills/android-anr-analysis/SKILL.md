---
name: android-anr-analysis
description: Diagnose Android ANRs (Application Not Responding) — read traces.txt/bugreports, locate the main-thread blocking frame, classify the root cause, and apply minimal off-main-thread fixes with the correct Coroutine dispatcher. Use when an app freezes, drops the "isn't responding" dialog, or shows sustained jank.
metadata:
  origin: ECC
version: 1.0.0
---

# Android ANR Analysis

Diagnose and fix Android ANRs. An ANR fires when the **main (UI) thread** cannot make progress before a system-enforced deadline. This skill covers how to read a trace, classify the root cause, and move the offending work off the main thread.

## When to Activate

- Reading an ANR `traces.txt`, an `anr_<timestamp>` file, or a bugreport's "VM TRACES" section
- Investigating freezes, frozen frames, or "App isn't responding" dialogs
- Diagnosing sustained jank ("Skipped N frames", "Slow operation/dispatch" in logcat)
- Deciding which Coroutine `Dispatcher` a piece of work belongs on
- Reviewing code for main-thread I/O, lock contention, or synchronous IPC before it ships

## ANR Types and Timeouts

| ANR type | Timeout | Triggered by |
|----------|---------|--------------|
| Input dispatching | 5s | Key/touch event not handled in time |
| BroadcastReceiver (foreground) | 10s | `onReceive` too slow |
| BroadcastReceiver (background) | 60s | `onReceive` too slow |
| Service (foreground) | 20s | `onCreate` / `onStartCommand` / `onBind` too slow |
| Service (background) | 200s | Service start too slow |
| ContentProvider | 10s | `publish` / query too slow |

## Where Traces Live

```bash
# On-device ANR dumps (root or run-as / debuggable build)
adb pull /data/anr/traces.txt            # legacy single-file
adb shell ls -t /data/anr/               # newer: anr_<pid>_<timestamp>

# System-recorded reason + slow-frame warnings
adb logcat -d | grep -iE 'ANR in|Reason:|Blocked|Slow (operation|dispatch)|Skipped [0-9]+ frames'

# Frame timing history (jank that precedes an ANR)
adb shell dumpsys gfxinfo <package>

# Bugreport: trace is under FS/data/anr/ or in the "VM TRACES JUST NOW" section
```

## How to Read the Trace

1. **Find the `"main"` thread** — it is `tid=1`, `prio=5`, named `main`. Only its blocking causes an ANR. Ignore blocked worker threads as the *cause* (they may be a clue to a lock holder).
2. **Read the top frames** — the first **application** frame beneath the blocking call is the suspect. Report it as `file:line` when symbols exist.
3. **Identify the wait primitive:**
   - `nativePollOnce` / `epollWait` on an otherwise **idle** stack → the main thread was likely **not** the culprit. The cause is usually CPU starvation, low memory, or another process. Recommend a Perfetto/systrace capture; do **not** "fix" idle app code.
   - `Object.wait` / `LockSupport.park` / "waiting to lock \<monitor\>" → lock contention or deadlock. Find which thread holds that monitor (`locked <monitor>`); if it is waiting on the main thread, you have a deadlock cycle.
   - `BinderProxy.transactNative` → synchronous IPC; the main thread is blocked on another process.
   - `read` / `write` / `SQLiteConnection.nativeExecute` / `SharedPreferencesImpl...commit` → blocking I/O.
   - `JSONObject.<init>` / `BitmapFactory.decode*` / `LayoutInflater.inflate` / large loops → heavy CPU work on main.

## Root Cause → Fix

| Root cause | Fix | Layer |
|------------|-----|-------|
| Blocking I/O on main | Wrap in `withContext(Dispatchers.IO)`; use `apply()` not `commit()`; do Room/SQLite off-main | Repository / DataSource |
| Lock contention / deadlock | Shrink the critical section; never hold a lock across a main-thread callback; break the wait cycle | wherever the lock lives |
| Synchronous Binder / IPC | Make the call async; cache results; avoid sync `ContentResolver`/AIDL calls on main | Repository |
| Heavy CPU work on main | Offload to `withContext(Dispatchers.Default)`; decode bitmaps in background; simplify the layout | ViewModel / DataSource |
| Message-queue congestion | Batch/throttle `post`s; debounce; drop `postDelayed` chains | UI |
| Lifecycle callback overrun | `BroadcastReceiver.goAsync()`, `WorkManager`, or `JobScheduler` instead of inline work | UI / framework |
| System-level (idle main) | Capture Perfetto/systrace; check memory and other processes — not an app code fix | — |

## The Dispatcher Rule (the heart of most fixes)

Keep the main thread for UI only. Push everything else to the right dispatcher:

```kotlin
// CPU-bound work (JSON parse, sorting, image decode) -> Default
withContext(Dispatchers.Default) { parseJson(largePayload) }

// Blocking I/O (disk, network, DB) -> IO
withContext(Dispatchers.IO) { database.query() }

// Touch the UI -> Main (default inside viewModelScope)
withContext(Dispatchers.Main) { updateUi(result) }
```

> See `skill: kotlin-coroutines-flows` for dispatcher details (including KMP, where `Dispatchers.IO` is JVM/Android-only) and `skill: android-clean-architecture` for which layer each call belongs in.

## Common Fix Patterns

```kotlin
// SharedPreferences: commit() blocks the caller; apply() is async
prefs.edit().putString("token", token).apply()   // not .commit()

// Move a synchronous DB read off the main thread
suspend fun loadUser(id: String): User = withContext(Dispatchers.IO) {
    userDao.findById(id)
}

// BroadcastReceiver: don't do real work inline
override fun onReceive(context: Context, intent: Intent) {
    val pending = goAsync()
    scope.launch(Dispatchers.IO) {
        try { handle(intent) } finally { pending.finish() }
    }
}
```

## Anti-Patterns to Avoid

- Doing disk/network/DB work, or `SharedPreferences.commit()`, on the main thread.
- Holding a lock while calling back into UI or framework code.
- Synchronous `ContentResolver`/Binder calls from `onCreate`/`onReceive`.
- "Fixing" an idle-main-thread ANR by changing app code — investigate system/CPU/memory first.
- Blaming a blocked worker thread; only the main thread's blocking triggers an ANR.

## Verification

After a fix, confirm the main thread no longer blocks:

```bash
adb shell dumpsys gfxinfo <package> | grep -iE 'Janky|frames|histogram'
adb logcat -d | grep -iE 'Skipped [0-9]+ frames|Slow operation'   # should disappear / shrink
```

Use StrictMode in debug builds to catch main-thread violations before they become ANRs:

```kotlin
StrictMode.setThreadPolicy(
    StrictMode.ThreadPolicy.Builder().detectDiskReads().detectDiskWrites().detectNetwork().penaltyLog().build()
)
```

## References

- Android docs: ANRs, StrictMode, App startup, WorkManager
- Tools: Perfetto / systrace, Android Studio Profiler, `dumpsys gfxinfo`
- Related skills: `kotlin-coroutines-flows`, `android-clean-architecture`
- Related agent: `android-anr-analyzer`
