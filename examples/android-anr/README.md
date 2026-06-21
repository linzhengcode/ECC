# Android ANR Example — Worked Analysis

A realistic, anonymized ANR capture used as a **few-shot exemplar** for the
`android-anr-analyzer` agent and the `android-anr-analysis` skill. It pairs raw
input (a trace + logcat) with the exact analysis and output the agent should
produce, so the model has a grounded reference for the workflow.

## Files

| File | What it is |
|------|------------|
| `traces.txt` | Authoritative ART thread dump from `/data/anr/traces.txt` (process `com.example.shop`, pid 12345) |
| `logcat-excerpt.txt` | The corresponding system-side ANR reason and slow-operation warnings |

## The Scenario

The user tapped "Add to cart". The screen froze and Android raised the
"isn't responding" dialog. We have the trace and a logcat excerpt.

## Worked Analysis (expected agent reasoning)

1. **ANR type + timeout** — `logcat-excerpt.txt` says
   `Reason: Input dispatching timed out ... 5009.1ms`. This is an **input
   dispatching ANR** (5s deadline exceeded).
2. **Find the `main` thread** — in `traces.txt`, `"main" prio=5 tid=1 Native`
   is the only thread whose blocking can cause an ANR.
3. **Read the top app frame** — beneath the `fsync` native frame and the
   SQLite/Room framework frames, the first **application** frame is:
   `com.example.shop.data.CartStore.persist(CartStore.kt:88)`, called from
   `CartViewModel.onAddToCart` → `CartActivity.onCreate(CartActivity.kt:96)`.
4. **What is it waiting on?** — the native stack is in `libc.so (fsync)` under
   `SQLiteConnection.nativeExecuteForChangedRowCount`: a **synchronous database
   write on the main thread**. `logcat` confirms it:
   `SQLiteStatement: Slow operation: 4123ms ... INSERT OR REPLACE INTO cart_items`
   and `Choreographer: Skipped 312 frames`.
5. **Cross-check other threads** — the `OkHttp Dispatcher` thread (tid=18) is
   running a coroutine `CartStore$persist$1` doing network I/O. The main thread
   is **not** waiting on a lock it holds (no monitor contention), so this is a
   blocking-I/O ANR, not a deadlock. The worker thread is a red herring for the
   *cause* but a clue that `persist()` mixes a main-thread DB write with a
   background fetch.
6. **Root cause** — Main-thread blocking I/O: `CartStore.persist()` performs a
   Room/SQLite write directly on the main thread from `onCreate`.
7. **Fix + layer** — move the write off the main thread (data / Repository
   layer) with `Dispatchers.IO`; never call DAO write methods synchronously
   from a UI callback.

## Expected Agent Output

```text
[ANR] Type: Input dispatching (5s exceeded; 5009ms elapsed)
Process: com.example.shop (pid 12345)
Blocking point: com/example/shop/data/CartStore.kt:88  (Room/SQLite write on main)
Root cause: Main-thread blocking I/O — synchronous DAO write (CartDao.upsert) called from onCreate
Evidence: main(tid=1) native fsync under SQLiteConnection.nativeExecuteForChangedRowCount;
          logcat "Slow operation: 4123ms ... INSERT OR REPLACE INTO cart_items"; "Skipped 312 frames"
Fix: Make persist() suspend and wrap the DAO call in withContext(Dispatchers.IO);
     call it from viewModelScope, not from Activity.onCreate
Layer: data (Repository / DAO)
Confidence: high

ANR Type: Input dispatching | Root Cause: Main-thread blocking I/O | Blocking: CartStore.kt:88 | Confidence: high
```

## The Fix (for reference)

```kotlin
// Before — blocks the main thread inside the Activity lifecycle
class CartStore(private val dao: CartDao) {
    fun persist(item: CartItem) {
        dao.upsert(item)            // synchronous SQLite write on main
    }
}

// After — write happens on Dispatchers.IO, driven from viewModelScope
class CartStore(private val dao: CartDao) {
    suspend fun persist(item: CartItem) = withContext(Dispatchers.IO) {
        dao.upsert(item)
    }
}

// CartViewModel
fun onAddToCart(item: CartItem) = viewModelScope.launch {
    cartStore.persist(item)         // UI thread stays free
}
```

See the `android-anr-analysis` skill for the full root-cause table and the
`android-anr-analyzer` agent for the diagnostic workflow.
