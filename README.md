# Firefox ESR — Moxfield UI Freeze (Gecko/WebRender)

## Related Issue

[webcompat/web-bugs#238716](https://github.com/webcompat/web-bugs/issues/238716)

## Symptoms

- Firefox becomes unresponsive while interacting with or refreshing Moxfield.
- The affected Firefox process can reach **100% CPU usage**.
- Keyboard shortcuts and other user interactions stop responding.
- The freeze occurs in Firefox's native **Gecko/WebRender rendering code**, rather than directly inside Moxfield's JavaScript.
- The main thread is occupied with hash-table iteration during a painting operation.
- The lack of keyboard responsiveness is consistent with main-thread blockage.
- Disabling hardware acceleration does not eliminate the affected native rendering path.

**Important:** The profiling evidence does not establish whether Moxfield's page content, a browser extension, or an internal Firefox defect originally triggered the condition.

## Technical Analysis

### 1. Initial Profiling

The original Firefox performance profile identified the following rendering call chain:

```text
GeckoMain
 └─ PVsync::Msg_Notify
    └─ RefreshDriver tick
       └─ Update rendering / Paint
          └─ nsLayoutUtils::PaintFrame
             └─ nsDisplayList::PaintRoot
                └─ WebRender display list
                   └─ libxul.so [unsymbolicated]
```

The initial profile pointed to Firefox's rendering pipeline but lacked sufficient symbol information to identify the specific internal operations consuming CPU time.

### 2. Re-Symbolicated Profile

After re-symbolication, the performance profile revealed a more precise call stack:

```text
nsRefreshDriver::PaintIfNeeded()
 └─ nsLayoutUtils::PaintFrame()
    └─ nsDisplayList::PaintRoot()
       └─ WebRenderCommandBuilder::BuildWebRenderCommands()
          └─ RemoveUnusedAndResetWebRenderUserData()
             └─ nsTBaseHashSet::RemoveIf()
                └─ nsTHashtableKeyIterator::operator++()
                   └─ PLDHashTable::Iterator::Next()
                      └─ PLDHashTable::Iterator::MoveToNextLiveEntry()
```

### 3. Suspected Bottleneck

The re-symbolicated profile provides evidence of a performance bottleneck in Firefox's WebRender user-data cleanup path.

The relevant operations are:

1. **Rendering:** `nsRefreshDriver::PaintIfNeeded()` initiates a painting operation.
2. **Display-list construction:** `WebRenderCommandBuilder::BuildWebRenderCommands()` prepares rendering commands.
3. **User-data cleanup:** `RemoveUnusedAndResetWebRenderUserData()` processes WebRender-associated data.
4. **Hash-table filtering:** `nsTBaseHashSet::RemoveIf()` iterates through entries for potential removal.
5. **Hash-table traversal:** `PLDHashTable::Iterator::MoveToNextLiveEntry()` appears in the CPU-intensive execution path.

These observations suggest that excessive processing during hash-table traversal or cleanup may be responsible for the main-thread stall.

However, the profile alone does not prove whether the underlying cause is an unusually large data structure, pathological iteration behavior, excessive repeated cleanup, or another upstream rendering condition.

## Impact

When the main thread is blocked, Firefox may fail to process:

- Keyboard input and shortcuts.
- Mouse interactions.
- Page refresh requests.
- Other main-thread browser operations.

The issue persists with hardware acceleration disabled, indicating that disabling GPU acceleration alone is insufficient to avoid the affected code path.

## Fix and Build Verification

A proposed fix, together with build and verification instructions, is documented here:

**[Fix download and Firefox ESR build verification guide](https://chatgpt.com/share/6adf294e-3a50-83eb-8112-2c182eebb99e)**

The linked instructions should be used to obtain the proposed changes, build Firefox ESR, and evaluate whether the modification resolves the freeze.

The proposed fix should be considered experimental until its behavior is verified against a reproducible baseline.

## Status

**Investigation:** Native rendering bottleneck identified through Firefox performance profiling.

**Root cause:** Not conclusively established.

**Fix:** Proposed; validation required.

**Affected components:** Gecko, WebRender, display-list construction, and hash-table iteration.

## Status Update

Navigating Moxfield with the [CPUs that affect the users experience behavior](https://github.com/scripting-drafts/CPU-Stress-Tests/tree/main) display the following results:

| Property | CPU 2 crash | CPU 8 crash |
|---|---|---|
| **Process** | Isolated Web Content | Isolated Web Content |
| **Faulting function** | `SnowWhiteKiller::Visit` | `CustomEvent_Binding::get_detail` |
| **Subsystem** | Cycle collector | DOM / JavaScript |
| **Fault type** | General protection fault | Invalid memory write |
| **Last page** | `/decks` | `/playtest` |
| **Physical core** | 2 | 2 |

Without the mentioned CPUs, it is confirmed (19.56 27 Oct) that Moxfield can drive heavy parallel rendering activity without causing Firefox to hang.

```text
taskset -c 0,1,3-7,9-11 firefox-esr --no-remote
```

```text
Average CPU: 108.1%
Peak over 1 second: 246.2%
Capture duration: 5.82 s
```

### Available next steps for testing Moxfield on CPUs that don't trigger Cycle Collector nor DOM / Javascript faults
 - Finding the exact threads using the CPU
 - Refining a Firefox Profiler values
 - Testing wether DOM mutations are excessive
 - Running controlled isolation tests
