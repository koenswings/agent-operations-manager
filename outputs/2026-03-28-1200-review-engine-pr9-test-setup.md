> **Task/Question:** Quality review of agent-engine-dev PR #9 — `test/pr1-disk-simulation`
> Cross-agent task from Axle: "Review the test setup design in PR #9. Verify that automated tests cover disk dock/undock and multi-engine scenarios. Raise concerns as PR comments."

## Council Review — Four Perspectives Synthesised

**PR:** `test/pr1-disk-simulation` (agent-engine-dev #9)
**Scope:** PR 1 of 6 — disk simulation infrastructure only (not multi-engine; that's PRs 3–4)
**Files reviewed:** `Config.ts`, `usbDeviceMonitor.ts`, `diskSim.ts`, `disk-dock-undock.test.ts`, `META.yaml`, `test-setup-native.md`

---

## Verdict: Approve with one blocker fixed

The design is sound and the implementation is clean. One bug prevents the tests from working as written. Everything else is solid or minor.

---

## Blocker — Must Fix Before Merge

**`TEST_DEVICE = 'sdtest1'` fails the `validDevice()` filter.**

`validDevice()` accepts `sd[a-z][1-2]` (e.g. `sda1`) or `sd[a-z]` (e.g. `sda`). The string `sdtest1` matches neither pattern. Chokidar fires `addDevice('/dev/engine/sdtest1')`, which extracts `sdtest1` as the device name — and then the function returns immediately with "not on a supported device name". The disk is never registered. All three tests will time out waiting for the store to update.

**Fix:** Change `TEST_DEVICE` in `diskSim.ts` to a name that passes the filter. Recommended: `sdz1` (seven characters but matches `sd[a-z][1-2]` — `s`, `d`, `z`, `1`). Using `sda` or `sda1` also works; `sdz1` is marginally safer since `sda` could theoretically be a real system disk on some test machines (though testMode skips mounting, so there's no actual risk).

```typescript
// diskSim.ts — change line:
export const TEST_DEVICE = 'sdtest1'
// to:
export const TEST_DEVICE = 'sdz1'
```

---

## Issues — Should Fix

**1. `IDEA_TEST_MODE` env var is set after `Config.ts` is imported.**

The pattern is:
```typescript
export const config = readConfig('./config.yaml');  // executed at module load
if (process.env.IDEA_TEST_MODE === 'true') {
    config.settings.testMode = true;               // mutates the already-exported object
}
```

This works only because `testMode` is read inside function bodies (not at import time in other modules). It is fragile: if any module ever reads `config.settings.testMode` during its own module-level initialisation (before the env override block runs), it will get `false`. The current code is safe, but this should be noted as a known constraint — ideally with a comment in Config.ts.

Suggested comment:
```typescript
// NOTE: This override runs at module load time. It is safe only if all consumers
// read config.settings.testMode inside function/method bodies, never at module scope.
```

**2. Harness creates `/disks/` and `/dev/engine/` but may lack permissions in some CI environments.**

`dockFixture()` calls `fs.ensureDir(DISKS_ROOT)` and `fs.ensureDir(DEV_ROOT)`. On a standard dev machine this requires write access to `/disks` and `/dev/engine`. These paths don't exist by default and may require sudo or pre-provisioning in CI. The `README` or test prerequisites section should document this explicitly.

**3. `test-setup-native.md` design doc not updated to `Implemented`.**

Status still reads: "Approved — implementation authorised. Update to `Implemented` when authoritative docs are updated." This PR is the first implementation step — update the status to `Partially Implemented (PR 1 of 6)` so future sessions know where we stand.

---

## Notes — Minor / Optional

**Design doc vs. implementation naming mismatch.**
The design doc shows helper names `dockDisk()` and `undockDisk()` in its code snippets. The actual harness uses `dockFixture()` and `triggerUndock()`. The names in the implementation are better (more precise), but the design doc should be updated to match before PR 2 lands.

**Test 2 and 3 depend on Test 1's store state.**
The three tests are sequentially dependent: Test 1 must succeed for Test 2 to have a disk to undock, and Test 3 re-docks after undock. This is acceptable given the stateful nature of dock/undock, but a comment in the test file noting the dependency would help future readers.

**Scope note: multi-engine not in this PR.**
The cross-agent task asked to verify "multi-engine scenarios". PR 9 is PR 1 of 6 — multi-engine is design-documented (PRs 3–4) but not yet implemented. This review confirms that the PR 1 scope (single-disk, single-engine, automated testMode) is correctly executed. The multi-engine coverage question should be revisited at PR 3.

---

## What's Done Well

- **Design doc (`test-setup-native.md`) is excellent.** Tier structure, diagnostic mode, field use, open questions, implementation plan — thorough and honest. The open questions are exactly the right ones.
- **`testMode` flag architecture is right.** Single flag, well-typed, threaded through the right places, env var override for CI. The production code path is not degraded.
- **Harness is clean and reusable.** `diskSim.ts` has focused, well-named, well-documented functions. Each does one thing. The `waitFor()` polling pattern is the correct approach for async state assertions.
- **Sentinel mechanism is honest.** Tests exercise the real chokidar watcher and the real `addDevice()`/`removeDevice()` paths. Only `sudo mount/umount` is bypassed — nothing else is mocked.
- **`IDEA_TEST_MODE=true` is set correctly in the npm scripts.** `test:unit` and `test:full` both set it.
- **Fixture is minimal and honest.** Real `META.yaml` structure, correct field names, minimal content. No unnecessary complexity.

---

## Summary

Fix `TEST_DEVICE = 'sdtest1'` → `'sdz1'` (or any valid `sd[a-z]` / `sd[a-z][1-2]` name). Add the Config.ts comment. Update the design doc status. Everything else can be addressed in subsequent PRs or as minor follow-up.

Do not merge until the blocker is resolved. Once fixed, this is a solid first PR.
