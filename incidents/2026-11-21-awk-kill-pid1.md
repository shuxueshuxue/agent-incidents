# An agent's awk tree walk put PID 1 in a root `kill` list

**Date:** 2026-11-21, 13:05:53 UTC
**Agent:** a Claude Code session running a UI-performance measurement lane
**Machine:** a Linux container shared by many agent lanes, each under its own non-root user, with `/tmp` on tmpfs
**Damage:** the container restarted. Its `/tmp` was emptied: every lane's working slot (checkouts, running app instances, unsaved evidence, test-account sign-in files) and the shared package store and slot template that new slots are built from. Persistent volumes survived. No production system was involved.

## What happened

The agent had finished a baseline run and wanted to stop the app instance it had launched (an Electron app, plus its children). It knew the instance's root pid. To catch every child, it walked the process tree of its lane user with awk, then killed the result. It ran this over SSH as root. Paths and the user name are shortened here:

```sh
P=$(python3 -c "import json;print(json.load(open('<slot>/instance.json'))['pid'])")
T=$(ps -u <lane user> -o pid=,ppid=,args= | awk -v r=$P '
  BEGIN { k[r]=1 }
  { p[$1]=$2; a[$1]=$0 }
  END {
    ch=1
    while (ch) { ch=0; for (i in p) if (k[p[i]] && !k[i]) { k[i]=1; ch=1 } }
    for (i in k) print i
  }')
echo $T | wc -w
kill $T 2>/dev/null
```

The intent: start from the root pid, keep adding any process whose parent is already in `k`, and print `k`.

The bug is the test `k[p[i]]`. In awk, reading an element that does not exist creates it, with an empty value. Each pass of the loop asked about the parent of every process the lane user owned, so every one of those parents became a key in `k`. They were never marked as children, but `for (i in k)` prints every key, set or not. Some of the lane user's processes had been started detached and were reparented to init, so their parent was PID 1, the container's supervisor.

The list printed 14 pids. `kill` ran as root, so nothing stopped it from signalling PID 1, and `2>/dev/null` hid what it did. The SSH connection dropped while the command was still running:

```
root=84839
14
Connection to 127.0.0.1 closed by remote host.
```

The membership test that does not create keys is `(p[i] in k)`.

## Why the damage was so wide

When PID 1 exits, the container restarts. `/tmp` was a tmpfs, so a restart empties it. Over the previous weeks it had become the place for everything a lane needed while working: one directory per lane, plus a shared read-only package store and a template that new lane directories are copied from. The store and the template are what make creating a lane directory cheap, and without them no new lane directory can be made. Work that had already been copied to the persistent volume (handoff bundles, archives, a git mirror) survived.

## Timeline (UTC)

| Time | Event |
|---|---|
| 13:05:53 | The agent issues the command. |
| 13:05:59 | The SSH connection is closed by the remote host. |
| 13:06:03 | The next SSH attempt is refused at the tunnel. |
| 13:06:52 | The supervisor starts again as PID 1 (from `ps -o lstart`). `/tmp` holds 428 KB. |
| 13:07 | SSH answers once. The agent checks: its lane directory and every other lane's are gone. |
| 13:08:41 | The agent reports the incident to its coordinator unprompted: the cause, what was lost and what survived, and the sign-in files that now need revoking. |
| 13:09–13:10 | Other lanes on the machine report their directories gone, and one reports its SSH key refused. The coordinator tells the owner, the lanes, and the agent that manages test accounts. |
| 13:12 | SSH is stable again. `/tmp` is still empty, and the package store and template are not yet rebuilt. |
| 13:13 | The test-account agent starts a global sign-out of the 46 accounts that had sign-in files on the machine during the past five days. A wiped file is lost, not leaked, so this is cleanup. |

## Who noticed

The agent that ran the command noticed first: its own SSH session died in the middle of a `kill`. It checked the supervisor's start time, connected that to its kill list, and found the awk behaviour before anyone asked. Other lanes noticed independently within a few minutes, when their directories were gone.

## Aftermath

The lane had already copied its baseline measurements off the machine before the kill, so its design work continued. Other lanes lost whatever had not been copied off and re-run only what their pull requests still needed, on another machine until the shared store is rebuilt.

## No new rule

The coordinator's first reaction was to give every lane new rules: no root kills, stop only pids recorded at launch, keep off the machine. The maintainer withdrew them within minutes: 「不要因为此次行为而惩罚或者约束谁 / 千万不要因此加护栏 / 保持狂野」 ("don't punish or restrict anyone because of this; absolutely no guardrails because of it; stay wild"). This entry records what happened. It adds no rule.
