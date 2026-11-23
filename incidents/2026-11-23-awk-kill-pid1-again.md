# The same awk tree walk restarted the container twice more, two days later

**Date:** 2026-11-23, 01:00:12 and 01:41:24 UTC
**Agent:** a Claude Code session fixing a small UI bug (a hover card with blank space after the name)
**Machine:** the same shared Linux container as [2026-11-21](2026-11-21-awk-kill-pid1.md), `/tmp` on tmpfs
**Damage:** the container restarted twice, 41 minutes apart. Each time `/tmp` was emptied: every lane's working slot (checkouts, builds, running app instances, half-finished check runs, test-account sign-in files) and the shared package store and slot template. Persistent volumes survived. No commits that had been pushed were lost. No production system was involved.

## What happened

The agent wanted to stop work it had started: at about 01:00 a full pre-push check run, at 01:41 two Electron instances it had launched for a before/after comparison. Both times it walked the process tree with an awk snippet of the same shape as the one in the 2026-11-21 entry and passed the result to `kill`, running as root:

```sh
ps -e -o pid=,ppid= | awk -v r=$ROOT '
  { p[$1]=$2 }
  END { a[r]=1; ch=1
        while (ch) { ch=0; for (k in p) if (a[p[k]] && !a[k]) { a[k]=1; ch=1 } }
        for (k in a) print k }' | xargs kill
```

The bug is the same: `a[p[k]]` creates a key for every parent it looks up, and `for (k in a)` prints every key. With `ps -e` the walk covered the whole container, so the list held every parent pid on the machine, including 0 and 1. `kill` as root signalled PID 1 (the container's supervisor), which logged `received SIGTERM indicating exit request` and exited, and the platform restarted the container.

The agent did not know about the earlier incident. It had not been told, and the snippet looks right on a quick read.

## How it was noticed and diagnosed

| Time (UTC) | Event |
|---|---|
| 01:00:12 | Supervisor receives SIGTERM; container restarts (`supervisord started with pid 1` at 01:00:48). |
| 01:10–01:20 | The lanes on the machine find their `/tmp` slots gone and report what they lost. The coordinator tells every lane to wait for the shared store to be rebuilt. A reverse SSH tunnel into the container also stays down: the container's `authorized_keys` is rewritten on restart, and the tunnel script tried other agent keys first and hit `Too many authentication failures`. |
| 01:17 | The shared store and template are rebuilt by hand. |
| 01:41:24 | Second SIGTERM to the supervisor; second restart. |
| 01:47 | Another coordinator adds a boot-time job that rebuilds the store and template automatically (about 3 minutes). It also asks whether the restarts were resource kills; load had reached 53 earlier. |
| ~01:48 | The machine's coordinator reads the supervisor log, sees a clean external SIGTERM both times (not an out-of-memory kill), and wrongly tells the others that the platform restarted the container. |
| 01:50 | The agent that ran the kills reports, unprompted, that both restarts were its own: it lined up its two kill times with the two restarts and reproduced the list with a dry run (printing only), which contained 0, 1 and 51. The coordinator corrects its earlier claim to the owner and the other coordinator. |

## Why it happened again

The 2026-11-21 entry was written down, but nothing carried it to the agents working on the machine: the next agent wrote the same snippet from scratch and had no reason to doubt it. Root access, `/tmp` holding all working state, and a supervisor as PID 1 were all unchanged.

## Aftermath

Every lane re-ran the checks that had been cut off. Several lanes changed their check runs to write each step's result to a persistent volume so that a restart resumes instead of starting over. A small `kill-tree.sh` (walks children with `pgrep -P`, prints the list, refuses pid 1 and the caller's own ancestors) was put next to the other lane scripts as a tool anyone may use; per the maintainer's standing word on the earlier incident, no rule was added.
