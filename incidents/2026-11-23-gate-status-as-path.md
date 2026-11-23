# An agent used a slot-pool status line as a path in `rm -rf`

**Date:** 2026-11-23, about 09:28 UTC
**Agent:** a Claude Code session running a document-compilation lane
**Machine:** a macOS build machine shared by many agent lanes, whose work directories are leased from a pool of numbered slots
**Damage:** Almost certainly none. `rm -rf` ran on four relative paths in the shared account's home directory, and none of them exists there afterwards. Nothing shows they existed before, but because the command had already run, that cannot be proven.

## What happened

Lanes on the machine borrow a work directory with a lease command (`gate acquire <key>`). When a slot is free, its last output line is the slot's absolute path. The agent's helper script relied on that:

```sh
G=$(ssh <machine> "gate acquire <key>" 2>&1 | tail -1)
```

This time every slot was taken. The lease command printed a warning, then the status table of all slots, and exited. The last line of that table belonged to another lane's slot:

```
g16  ready  <other lane's key> 7min
```

`G` became `g16 ready <other lane's key> 7min`. The script did not check the value. The compile steps that used it failed with argument errors, and then the cleanup step ran, unquoted, in the account's home directory over SSH:

```sh
rm -rf $G/anh && gate release $G <key>
```

which expanded to

```sh
rm -rf g16 ready <other lane's key> 7min/anh && gate release g16 ready <other lane's key> 7min <key>
```

The four relative targets were words from a status table, not directories the agent had created. The `release` that followed was refused ("not a pool directory"), so no other lane's lease was released.

## How it was noticed

The agent saw the failed compile output, the refused `release` and `ls` errors for the four names in its own command output, and recognized the status-table line in `$G`.

## How it was checked

- `ls -d` on the four names in the home directory: none exists.
- A search of the home directory to depth 3 found no directory with the other lane's key in its name.
- The real slot directory that line referred to was present and still leased by its owner.
- Shortly afterwards, the lane's supervisor checked the machine independently and reached the same result.

Conclusion: almost certainly no damage. A directory with one of these names could only have been lost if it existed before the command ran, and no record of the home directory from before then exists.

## Timeline (UTC)

| Time | Event |
|---|---|
| about 09:27 | The lease command finds no free slot and prints the status table; the helper takes its last line as the slot path. |
| about 09:28 | The cleanup line runs `rm -rf` on the four relative paths; the following `release` is refused. |
| a few minutes later | The agent stops, inspects the home directory and the slot pool, adds the path assertion below, and reports the incident to its supervisor unprompted. Times in this row are approximate. |
| 09:36 | The supervisor's independent check finds the same state. |

## Repair

There was nothing to restore. The helper scripts were changed so that the slot value is checked before anything uses it:

```sh
[[ "$G" =~ ^/Users/<account>/gates/g[0-9]{2}$ ]] || { echo "bad slot: $G" >&2; exit 1; }
```

Acquisition now goes through a script that prints only a validated path and fails otherwise. Removal goes through a separate script that repeats the check, prints the exact path, quotes it, and only then runs `rm -rf` and the release. The lane retried later, got a free slot, and finished normally.

## What made it possible

The command's output format depends on state: a path when a slot is free, a table when none is. `tail -1` hides that difference. The `rm -rf` sat at the end of a command chain whose earlier steps had already failed, and nothing stopped it because it did not depend on their exit codes.
