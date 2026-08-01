# SM-S928W Porting Progress

**Device:** Samsung Galaxy S24 Ultra (Canadian variant, SM-S928W)
**Firmware:** BP4A.251205.006.S928WVLS6DZF2 (June 2026 security patch)
**Kernel:** 6.1.145-android14-11-33419968-abS928USQS6DZF2
**Date:** 2026-07-31

## Summary

Kernel binary confirmed byte-identical to existing e3q-S928USQS6DZF2 profile (BTF SHA-256 matches). All 20 symbol offsets, BTF struct layouts, and 256/256 P0 fingerprint qwords verified. Through parameter tuning, achieved the FIRST successful gate reclaim on S24 Ultra hardware.

## Verification Results

- 20/20 symbol offsets: MATCH
- All BTF struct layouts: MATCH
- P0 fingerprint table: 256/256 qwords MATCH
- BTF SHA-256: 8415104c...d65efe56 (identical to case study)
- SLIDE_TRACEFS_EVENT_ID: 106 (confirmed via ADB)
- SLIDE_TRACEFS_WORKER_CALLER_OFF: 0x000db1a0 (confirmed via disassembly)

## Tuning History

### Baseline (original parameters)
- SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS=2, no SLIDE_PHYSICAL_SLOT_DELAYS_USEC
- Result: All 24 attempts failed at pipe page leak or mm_struct leak
- Best: attempt 7 got pipe oracle prepared, phone rebooted during P0 oracle phase

### Tuning 1: verbose=1 logging
- Enabled KernelSnitch verbose output in util.c
- Result: Identified three failure modes (pipe page leak, mm_struct leak, gate miss)
- Best: attempt 2 got mm_struct leak + kernel page, but gate reclaim missed (hits=0)

### Tuning 2: SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS 2->8
- Result: attempt 23 achieved mm_struct leak on retry 3, kernel page prepared
- pselect race ran (ret=0, timing miss on probe)
- No reboot, all 24 attempts completed

### Tuning 3: SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS 8->16
- Result: attempt 1 achieved mm_struct leak on FIRST try, kernel page prepared
- pselect gate race SUCCEEDED (ret=2, ok=1)
- GATE RECLAIM SUCCEEDED (hits=1, changed=0) -- FIRST TIME ON S24 ULTRA
- Probe pselect race failed (ret=0, timing miss)
- Exploit aborted after 1 attempt (oracle dirtied, refusing unsafe retry)

### Tuning 4: SLIDE_PHYSICAL_SLOT_DELAYS_USEC (current)
- Added 8 delay values: 20000, 25000, 30000, 15000, 35000, 40000, 10000, 45000
- Result: Phone rebooted during attempt 2 bruteforce (probabilistic crash)
- Needs retry -- the crash happened before reaching the probe phase

## Key Findings

1. SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS=16 is critical -- the mm_struct leak is probabilistic and needs multiple retries
2. The gate pselect race works with delay=25000 (ret=2, gate hits=1)
3. The probe pselect race needs a DIFFERENT delay than the gate (ret=0 with same delay)
4. SLIDE_PHYSICAL_SLOT_DELAYS_USEC gives the probe multiple delay attempts
5. Phone reboots are probabilistic -- the KernelSnitch bruteforce touches physical memory

## Current Build Parameters

- src/common.h (APP_PAYLOAD): SLIDE_KERNEL_PAGE_SETUP_ATTEMPTS=16, FOPS_KERNEL_PAGE_SETUP_ATTEMPTS=16
- src/targets/e3q-S928USQS6DZF2/target.h: SLIDE_PHYSICAL_SLOT_DELAYS_USEC with 8 delays
- src/util.c: kernelsnitch_setup verbose=1

## Next Steps

1. Retry the exploit (probabilistic -- may succeed on next run without crash)
2. If probe still fails after multi-delay, try narrowing delays near 25000
3. Consider increasing EXPLOIT_ATTEMPTS beyond 24 if bruteforce crash rate is high
