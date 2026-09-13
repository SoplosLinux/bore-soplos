# bore-soplos — BORE CPU Scheduler patch for Soplos Linux

Rebase of firelzrd's BORE (Burst-Oriented Response Enhancer) scheduler patch,
kept building against current Soplos kernel point releases when upstream
hasn't caught up yet.

> Not yet wired into **soplos-kernel-installer** — `core/downloader.py` still
> fetches BORE from firelzrd/CachyOS upstream. This repo exists so a working
> patch is ready the moment it's needed, not to replace the upstream source
> by default. See "Status" below.

---

## What BORE does

BORE tracks each task's recent CPU "burst" time (how long it has run since
last sleeping) and converts that into a scheduling penalty: tasks that burst
less — interactive tasks like a compositor, a terminal, game input threads —
get prioritized over CPU-bound background tasks. The penalty decays over time
and is partly inherited by child/thread-group tasks so short-lived forks
don't restart at a neutral priority. It replaces EEVDF's tunable-scaling
defaults with a simpler constant-slice model (`CONFIG_MIN_BASE_SLICE_NS`).

Upstream project: [firelzrd/bore-scheduler](https://github.com/firelzrd/bore-scheduler).
Soplos does not modify this algorithm — only where necessary to keep the
patch applying against a newer kernel source tree (see below).

---

## Patch files

| File | Kernel versions | Base |
|------|-----------------|------|
| `patches/0001-bore-7.1.patch` | Linux 7.1.0 – 7.1.5 | BORE 6.6.3 |
| `patches/0001-bore-7.2.patch` | Linux 7.2 | BORE 6.8.0-rc1 |

`soplos-kernel-installer` requests the file matching the exact `major.minor`
kernel line — there is no generic fallback for BORE, unlike X3D/march.

---

## Why this rebase exists (7.1.5)

firelzrd publishes BORE against the `-rc1` snapshot of each kernel line
(`linux7.1-rc1-bore-6.6.3.patch`). Soplos ships stable point releases
(7.1.5), which accumulate upstream stable-tree backports on top of that rc1
base. Most of the time the drift is small enough that `patch -p1` applies
with a line-offset only.

For 7.1.5 specifically, a stable backport to `kernel/sched/fair.c` reworked
`util_est` handling (moved the `util_est_update()` call out of
`dequeue_task_fair()` into `update_load_avg()`, behind a new
`UPDATE_UTIL_EST` flag). BORE's `dequeue_task_fair()` hook
(`restart_burst_bore()` call) was textually anchored right after that now-gone
`util_est_update()` line, so of the patch's 21 hunks, exactly 1 failed —
everything else applied with offset only, no fuzz.

**Fix applied:** the hook was moved to anchor after `util_est_dequeue(&rq->cfs, p)`
instead (same function, a few lines earlier — the nearest still-existing
anchor). No change to BORE's own logic: same `update_curr()` /
`restart_burst_bore()` calls, same `(flags & DEQUEUE_SLEEP) && entity_is_task(se)`
guard, unmodified.

---

## Why this rebase exists (7.2)

Kernel 7.2 was released 2026-08-16. No upstream BORE source had a 7.2
release ready at the time — `soplos-kernel-installer`'s dry-run fallback
chain fell through to this repo, and the file did not exist yet, which
made the whole BORE patch silently unavailable on 7.2 until this rebase.

Unlike 7.1.5, this one needed **no fix at all** — verified applying clean
against real 7.2 sources with plain line-offset only (up to 2 lines in
`fair.c`), no fuzz, no failed hunks.

---

## Status

- **7.1.5:** verified with `patch -p1 --dry-run` (all 13 touched files
  clean, no fuzz, no rejects), **compiled and boot-tested on real
  hardware** — confirmed active via runtime sysctl checks, not just
  compiled in.
- **7.2:** verified with `patch -p1 --dry-run` (clean, no fuzz, no
  rejects), **compiled and boot-tested on real hardware**
  (`7.2.5-soplos-bore-ntsync-v3`, currently in production use); this
  validation has been ongoing across every Soplos kernel release since 7.1.
- Not verified against 7.1.0–7.1.4 (x3d-soplos's equivalent patch applies to
  the whole 7.1.x line with offset only; this one hasn't been checked the
  same way yet).

---

## Applying the patch manually

```bash
cd /path/to/linux-7.1.5
patch -p1 < /path/to/patches/0001-bore-7.1.patch

# or, for 7.2:
cd /path/to/linux-7.2
patch -p1 < /path/to/patches/0001-bore-7.2.patch
```

---

## Modified files

| File | Change |
|------|--------|
| `include/linux/sched.h` | `struct bore_ctx` embedded in `task_struct` |
| `include/linux/sched/bore.h` | New file — public API |
| `init/Kconfig` | `CONFIG_SCHED_BORE` option |
| `kernel/Kconfig.hz` | `CONFIG_MIN_BASE_SLICE_NS` option |
| `kernel/exit.c`, `kernel/fork.c` | Sibling-list handling for burst inheritance on fork/exit |
| `kernel/futex/waitwake.c` | Marks `bore.futex_waiting` around `schedule()` |
| `kernel/sched/Makefile` | Builds `bore.o` when `CONFIG_SCHED_BORE=y` |
| `kernel/sched/bore.c` | New file — burst penalty algorithm, inheritance caches |
| `kernel/sched/core.c` | `effective_prio_bore()` used in `set_load_weight()`; `sched_init_bore()` call |
| `kernel/sched/debug.c` | Extra sysctl/debugfs entries, `bore.score` in task dumps |
| `kernel/sched/fair.c` | Hooks in `update_curr`, `place_entity`, `requeue_delayed_entity`, `enqueue_task_fair`, `dequeue_task_fair`, `yield_task_fair`, `switched_to_fair` — this is the file with the 7.1.5-specific rebase described above |
| `kernel/sched/sched.h` | Extern declarations for the new sysctls |

---

## Kernel variants using this patch

Intended for `soplos-bore` and any variant combining `bore` with `ntsync`/`x3d`
(x3d always requires bore, per `soplos-kernel-installer`'s patch selector).
Built and boot-tested — see "Status".

---

## License

GPL-2.0 (inherited from the Linux kernel and firelzrd's original patch).
