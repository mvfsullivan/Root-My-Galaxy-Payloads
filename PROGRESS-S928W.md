# SM-S928W Porting Progress — Final Report for Dev Review

**Device:** Samsung Galaxy S24 Ultra (Canadian variant, SM-S928W)
**Firmware:** BP4A.251205.006.S928WVLS6DZF2 (June 2026 security patch)
**Kernel:** 6.1.145-android14-11-33419968-abS928USQS6DZF2
**Date:** 2026-08-01

## Executive Summary

Kernel binary confirmed byte-identical to e3q-S928USQS6DZF2 profile. All offsets verified. Through parameter tuning, achieved the FIRST and SECOND documented gate reclaims on S24 Ultra hardware. The probe pselect race remains the unsolved bottleneck — it requires a sub-microsecond timing window that no delay value we've tested can hit.

## What Worked

### 1. SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS=16 (was 2)
The mm_struct leak is probabilistic. With the original value of 2, it almost never succeeded. With 16, it succeeds on retry 3-9, reliably reaching the pselect gate phase. This is a critical fix for app-mode payloads.

### 2. Gate reclaim achieved (twice)
- Run 1 (APK, tuned2): attempt 1, delay=25000, object_index=4, pipe=24, gate hits=1
- Run 2 (local shell, 64 attempts): attempt 39, delay=45000, object_index=20, pipe=60, gate hits=1

Both achieved gate hits=1, changed=0 — the exact success condition. This is further than any documented attempt on S24 Ultra.

### 3. Shell context reduces reboots
Running from `adb shell` (uid=2000, u:r:shell:s0, Seccomp=0) instead of the APK (uid=10429, u:r:untrusted_app:s0, Seccomp=2) eliminated reboots in one 64-attempt run. However, reboots still occur intermittently — shell context helps but doesn't guarantee stability.

### 4. object_index correlation
- object_index=4: gate hit (run 1)
- object_index=20: gate hit (run 2)
- object_index=0: gate reclaim miss even when pselect ret=2
- object_index=16,24: various results

This suggests the page layout affects whether the physical write lands in the correct pipe buffer.

## What Didn't Work

### 1. SLIDE_PHYSICAL_SLOT_DELAYS_USEC (probe multi-delay)
Tested 8 delay values (20000-45000 and 45000-60000). When the gate succeeds and the probe tries all 8 delays, ALL return ret=0. The probe pselect race never hits the timing window regardless of delay.

Key observation: the gate succeeds with ret=2, but the probe with the same delay returns ret=0. The probe writes to a different physical offset (P0_ORACLE_PROBE_OFFSET=0x1f0000 vs gate's 0x0e80) — the timing window is different and none of our delays hit it.

### 2. PSELECT_ENTER_DELAY_USEC variations
Tested delays from 10000 to 60000 across many runs. The gate race succeeds ~3% per attempt. The probe race succeeds 0% across all runs.

## The Core Problem

The pselect race works by:
1. Consumer thread calls sched_setattr() to change the waiter's scheduling
2. Waiter thread is in pselect6() syscall
3. The sched_setattr must fire during the microsecond window when pselect copies fd_sets to the stack
4. The fake waiter (on the stack) gets overwritten with our controlled data

For the GATE: this works ~3% of the time — the timing window exists and is hittable.
For the PROBE: the timing window appears to not exist or is much narrower. All 8 delays return ret=0 (pselect returns 0 ready fds — the race never triggers).

This suggests the probe's physical page layout or scheduling path is fundamentally different from the gate's, and the current timing approach cannot hit it.

## Data for the Dev

### Gate hit logs (2 occurrences)
Both show: pselect ret=2, sched_ok=1, gate hits=1, changed=0
Full verbose logs available in the offline-testing-e3q branch.

### Probe failure logs (all attempts)
All show: pselect ret=0, sched_ok=1 (sched_setattr succeeds but race doesn't trigger)
The consumer thread fires correctly (sched_ok=1) but the pselect doesn't observe the fd state change.

### object_index data
- Gate success: index=4, index=20
- Gate miss with ret=2: index=16
- Gate miss with ret=0: various

## Current Build Configuration

- src/common.h: SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS=16, FOPS_KERNEL_PAGE_SETUP_ATTEMPTS=16
- src/targets/e3q-S928USQS6DZF2/target.h: SLIDE_PHYSICAL_SLOT_DELAYS_USEC with 8 delays (45000-centered)
- src/util.c: kernelsnitch_setup verbose=1
- Branch: offline-testing-e3q

## Questions for the Dev

1. Why does the probe pselect race consistently return ret=0 while the gate race can return ret=2? Is the timing window fundamentally different for P0_ORACLE_PROBE_SLOT vs P0_ORACLE_GATE_SLOT?

2. Could P0_ORACLE_PROBE_OFFSET (0x1f0000) be wrong for this kernel? The gate offset (0x0e80) works — is there a way to verify the probe offset?

3. Is there a diagnostic mode (like P0_ORACLE_GATE_DIAG) that can test the probe without proceeding to slide detection?

4. Could the CONSUMER_MAX_CALLS=1 limitation be the issue? The consumer fires sched_setattr once per child — would multiple calls per child help?

5. The object_index correlation (0 = miss, 4/20 = hit) — is this meaningful? Does the slab position affect the physical page alignment?

## Repository

- Fork: github.com/mvfsullivan/Root-My-Galaxy-Payloads
- Branch: offline-testing-e3q
- All verbose logs and tuning history committed
