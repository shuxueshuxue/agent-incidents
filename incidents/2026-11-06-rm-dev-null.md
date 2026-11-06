# An agent ran `rm -rf -- /dev/null` as root

**Date:** 2026-11-06, about 05:57 UTC
**Agent:** a Claude Code session running a test lane
**Machine:** a Linux box shared by many agent lanes, each mostly under its own non-root user
**Damage:** `/dev/null` was unusable for non-root users for about 50 seconds. No data was lost.

## What happened

The agent was starting an A/B build on the shared machine. It sent one long SSH command as root. Paths are shortened here:

```sh
nohup bash /tmp/ds-ab.sh > /tmp/ds-ab.log 2>&1 &
sleep 25
cat /tmp/ds-ab.log | cut -c1-120
O=<path to the baseline worktree>
rm -rf -- /dev/null
echo
ls <secrets dir> | grep <lane name>
```

Nothing in the task called for the `rm`. `$O` is set and never used, and the lines after it are unrelated. The session log records no reason for the line, and the agent later described it only as its own mistake when writing the command. Why it wrote that line is unknown.

`--` and an absolute path make this command look careful. It still did exactly what it said, and root is allowed to do it.

## Why it was visible

Deleting the device node was only the first step:

1. `rm` removed the character device.
2. Some process running as root then redirected output to `/dev/null`. With nothing at that path, the redirect created a regular file, mode 644, owned by root.
3. Every non-root redirect to `/dev/null` now failed with `Permission denied`. Root processes kept working and quietly appended to the file, which held 8 bytes by the time it was inspected.

So the failure only hit other users, and it surfaced far from its cause. The first symptom was in another lane's installer:

```
install.sh: 66: cannot create /dev/null: Permission denied
install.sh: 69: cannot create /dev/null: Permission denied
install.sh: 76: cannot create /dev/null: Permission denied
install.sh: 78: cannot create /dev/null: Permission denied
安装失败:需要 sha256sum 或 shasum 来校验下载的包
```

The last line says "install failed: sha256sum or shasum is required to verify the download". The tool was there: the installer's check for it had its output redirected to `/dev/null`, so the check failed, and the error blamed a missing checksum tool.

## Timeline (UTC)

| Time | Event |
|---|---|
| 05:56:54 | The agent issues the command. The `rm` comes after `sleep 25`, so it runs at about 05:57:20 (inferred). |
| 05:57:41 | Another agent's installer fails with the errors above. |
| 05:57:48 | That agent stops debugging its installer and inspects the machine: `/dev/null` is `-rw-r--r-- root root`, while `/dev/zero`, `/dev/urandom` and `/dev/tty` are fine. A non-root write returns rc=2. |
| 05:58:02 | It creates a replacement beside the damaged file: `mknod -m 666 /dev/null.sact c 1 3`. |
| 05:58:08 | It swaps the replacement in with `mv -f`, confirms a non-root write returns rc=0, and checks again 20 seconds later. |
| 05:58:57 | The agent that ran the `rm` finds the line in its own command and reports it unprompted. |

## Who repaired it

The agent that hit the symptom, which had not caused the problem and did not yet know who had. It read the machine's state instead of retrying its installer, and it built the new device beside the old file before swapping, so `/dev/null` was never missing during the repair.

## Aftermath

Red test results from the window were rerun against a baseline. Green results were kept, because a broken `/dev/null` could fail a run but not make one pass. Production and other environments were not affected.

## No new rule

The maintainer's reaction to a proposed set of new rules was that writing up incidents as rules makes agents timid and slows development, so the rules were rolled back. This repository keeps the record without adding a rule.
