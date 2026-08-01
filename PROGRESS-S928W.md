# SM-S928W Porting Progress

**Device:** Samsung Galaxy S24 Ultra (Canadian variant, SM-S928W)
**Firmware:** BP4A.251205.006.S928WVLS6DZF2 (June 2026 security patch)
**Kernel:** 6.1.145-android14-11-33419968-abS928USQS6DZF2
**Date:** 2026-07-31

## Summary

Kernel binary confirmed byte-identical to existing e3q-S928USQS6DZF2 profile (BTF SHA-256 matches). All 20 symbol offsets, BTF struct layouts, and 256/256 P0 fingerprint qwords verified. Exploit runs, gets past pipe page leak on attempt 7, then crashes during P0 oracle phase. This is the same "HW debugging in progress" issue — not an offset problem.

## Verification Results

- 20/20 symbol offsets: MATCH
- All BTF struct layouts (file_operations, task_struct, page, etc.): MATCH
- P0 fingerprint table: 256/256 qwords MATCH
- BTF SHA-256: 8415104c...d65efe56 (identical to case study)
- SLIDE_TRACEFS_EVENT_ID: 106 (confirmed via ADB)
- SLIDE_TRACEFS_WORKER_CALLER_OFF: 0x000db1a0 (confirmed via disassembly)

## Exploit Run (2026-07-31)

- Attempts 1-6: failed at KernelSnitch pipe page leak (expected — probabilistic)
- Attempt 7: BREAKTHROUGH — pipe page leak succeeded, P0 oracle prepared (base=ffffff8932150000, pipes=240)
- Phone rebooted during P0 oracle gate/probe trigger phase (kernel panic)

This is further than the dev documented. The dev got "reclaim misses" (silent failures); we got a crash at the oracle phase, meaning the oracle is attempting physical memory writes.

## Tuning Plan

Dev tested: PIPE_RECLAIM_SLABS (28/30/32), PIPE_DRAIN_SLABS (28-32)
NOT tested (our targets): PSELECT_ROUTE_NFDS, PSELECT_ENTER_DELAY_USEC, SLIDE_REQUEUE_MAX_POLLS, SLIDE_REQUEUE_POLL_USEC, SLIDE_PSELECT_TIMEOUT_NSEC, SLIDE_KSNITCH_* parameters

First step: enable verbose KernelSnitch logging to capture crash location.
