# Changelog — bore-soplos

## 1.1.0 — 2026-08-17

### Added

- `0001-bore-7.2.patch` — BORE 6.8.0-rc1 rebase for Linux 7.2. Applies clean
  against real 7.2 sources with plain line-offset only (up to 2 lines in
  `fair.c`), no fuzz, no failed hunks — no logic changes needed relative to
  the 7.1 line.

### Notes

- Verified with `patch -p1 --dry-run` against kernel.org tag `v7.2`, all 13
  touched files.
- Not build-tested, not boot-tested yet.
- Kernel 7.2 was released 2026-08-16; no upstream BORE source had a 7.2
  build ready at time of writing, which is what this rebase is for.

## 1.0.0 — 2026-07-25

Initial release.

### Added

- `0001-bore-7.1.patch` — rebase of firelzrd's BORE 6.6.3 patch
  (`patches/stable/linux-7.1-bore/0001-linux7.1-rc1-bore-6.6.3.patch`) so it
  applies against Linux 7.1.5.

### Fixed

- The `dequeue_task_fair()` hook in `kernel/sched/fair.c` was anchored on the
  `util_est_update(&rq->cfs, p, flags & DEQUEUE_SLEEP)` call, which the 7.1.5
  stable backport removed (moved into `update_load_avg()` behind
  `UPDATE_UTIL_EST`). Re-anchored the same hook (unmodified logic) after
  `util_est_dequeue(&rq->cfs, p)` instead. All other 20 hunks in the patch
  were unaffected (offset only, no fuzz).

### Notes

- Verified with `patch -p1 --dry-run` against kernel.org tag `v7.1.5`
  (stable branch), all 13 touched files.
- Not build-tested, not boot-tested. Do not package a kernel from this patch
  without doing both first.
- Not submitted upstream. Soplos Linux rebase of firelzrd/bore-scheduler.
