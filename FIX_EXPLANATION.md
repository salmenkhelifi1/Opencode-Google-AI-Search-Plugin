# Fix: Storage Leak & Auto-Recovery Mechanism

## The Problem
The `google-ai-search` plugin was previously launching a **new** headless Chromium browser instance for every single search request.

### Impact on System Resources:
1.  **Storage (Temporary Files):** Each browser session creates a temporary profile in `$TMPDIR` (approx. 50-100MB per launch). If 20 searches run, that's ~2GB of temp files.
2.  **Memory & Swap:** Keeping multiple browser processes or zombie processes can fill up RAM. When RAM is full, macOS writes "Swap" files to the disk (`/var/vm`), which can grow to multiple Gigabytes.
3.  **Performance:** Launching a browser from scratch takes 1-2 seconds, slowing down every search.

## The Solution

I have implemented a **Singleton Resource Manager** with **Auto-Recovery**.

### 1. Browser Reuse (Singleton Pattern)
Instead of `const browser = await launch()`, we now use a global `GoogleAIModeManager` instance.
- **First Request:** Launches the browser.
- **Subsequent Requests:** Reuses the *existing* open browser page.
- **Benefit:** Zero new temp files for subsequent searches. Instant response times.

### 2. Auto-Recovery (Idle Cleanup)
A timer tracks activity.
- **Logic:** `IDLE_TIMEOUT` is set to 5 minutes (300,000 ms).
- **On Request:** The timer is cleared.
- **On Completion:** The timer starts.
- **On Timeout:** If no new requests come in for 5 minutes, the `dispose()` method is called.

```typescript
// Simplified Logic
startIdleTimer() {
  this.idleTimer = setTimeout(() => {
    this.dispose(); // Closes browser & cleans up temp files
  }, 5 * 60 * 1000);
}
```

### 3. Concurrency Locking
Since we are sharing a single page, we cannot have two searches typing in the search bar at the exact same time.
- Implemented a `queryLock` (mutex) that queues requests if they happen simultaneously.

## How to Verify
1.  Monitor disk usage: `du -sh $TMPDIR/playwright*`
2.  Run a search: `google_ai_search_plus "test"`
3.  Wait 5 minutes.
4.  Check processes: `ps aux | grep Chromium` should be empty after the timeout.
