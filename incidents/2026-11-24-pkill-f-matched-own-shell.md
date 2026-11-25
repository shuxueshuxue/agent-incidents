# An agent building a guard against mistaken kills killed its own shell four times with `pkill -f` patterns

**Date:** 2026-11-24, from 16:57 UTC, in under half an hour
**Agent:** a Claude Code session whose task was to build a command-safety tool for agent harnesses: one that shows an agent what its command is really about to do when that differs from what it meant
**Machine:** a Linux host shared by many agent sessions under one user account
**Damage:** the agent's own tool-call shell was killed four times, taking the rest of each command with it. No other session lost anything. One command in the same series would have killed two processes belonging to other sessions; the tool under construction stopped it.

## What happened

To test the tool, the agent started throwaway `sleep` processes and then cleaned them up by pattern. Every one of these commands had the same flaw, and the agent wrote it five times.

`pgrep -f` and `pkill -f` match against the whole command line of every process. The harness runs each tool call as `bash -c '<the entire command text>'`. So the shell executing the command always has the pattern on its own command line, and matches itself.

Numbered by kind; in time the order was 1, 2, 4, 3, 5.

**1. Inside the tool, not stopped. Shell killed.** The first smoke test, at 16:57:41:

```sh
<tool> run -c 'nohup sleep 300 >/dev/null 2>&1 & echo started'
<tool> run -c 'kill $(pgrep -f "sleep 300")'
```

Both lines were part of one tool call, so the outer shell's command line contained `sleep 300`. `pgrep` returned the `sleep`, the inner shell, and the outer shell. The tool was supposed to see this `kill` and did not: it routed the shell's `kill` through a function that called `command kill`, and in bash `command kill` still runs the builtin. Nothing looked at the pid list. The tool call ended with exit code 144 and no output after that line.

**2. Inside the tool, stopped.** After fixing the routing, the agent reran a similar test:

```sh
<tool> run -c 'sleep 555 & echo bg=$!'
<tool> run -c 'pkill -f "sleep 55[5]"; echo after-ok'
```

The bracket is the usual trick to keep `pgrep` from matching itself. It does nothing for the shell that carries the pattern as text. This time the tool reported:

```
STOPPED. The command was cut off before this took effect:
  pkill -f 'sleep 55[5]'
    would signal 2 processes
    ! 1 of 2 targets were not started by this session: 2463042 /bin/bash -c source <snapshot path>…
```

Pid 2463042 was the outer shell. This was the first moment the agent understood what had happened in step 1. It had read that exit as a harness hiccup.

In the same run, another test line, `kill $(pgrep -x sleep -u <user> | head -50)`, matched two `sleep 30` processes that the agent had not started; they belonged to other sessions on the machine. The tool stopped that one too. Without it they would have been killed.

**3. Outside the tool. Shell killed.** Between the two script runs below, the agent cleaned up leftover test processes by hand, in a plain tool call:

```sh
for p in $(pgrep -f 'sleep 4321[7]') $(pgrep -f 'sleep 3077[1]'); do echo "mine: $p ..."; kill $p; done
... sed -i 's/sleep 30771 & echo started/.../' test/incidents.sh
```

The same tool call contained a `sed` whose text mentioned `sleep 30771`. The loop printed three real leftovers, then printed its own shell as `mine:`, then killed it. The label was the agent's own assertion; nothing had checked it.

**4 and 5. Outside the tool, in the tool's own test script. Shell killed twice.** The script that replays kill incidents had this setup line:

```sh
sleep 0.3; victim=$(pgrep -f 'sleep 4321[7]' | head -1)
...
kill "$victim" 2>/dev/null
```

`head -1` takes the lowest pid. When the script was launched from a tool call whose text also contained that pattern (the agent was editing the pattern in the same command), the lowest matching pid was the outer shell. The first time, the agent could not explain the exit and reran the script detached, where it passed. The second time it found the line.

## Who noticed

For 1, 3, 4 and 5 the only signal was the tool call returning exit code 144 with its output cut short. There was no error message, because the process that would have printed one was the one killed. The agent explained the first one away and the fourth one as a mystery. It was the tool's report in step 2, naming the outer shell as a target, that identified the cause, and the `mine:` line in step 3 that showed it again in plain sight.

## Repair

Nothing needed restoring: each kill cost one tool call, which the agent reran.

Two things were changed.

- The tool: the shell function that routes `kill` now calls the stand-in by absolute path instead of `command kill`.
- The test script: the victim's pid is taken from `$!` at the point it is started, not searched for by pattern afterwards. The script carries a comment saying why.

## Why this is here

The agent's whole task that day was this class of mistake. It had read the earlier entries in this repository, and its own notes from a previous incident say to stop processes only by pids recorded at launch. It still wrote the pattern search five times in under half an hour, twice in the test for the very tool meant to catch it. Each command looked correct as text, and four of the five carried the `[x]` bracket that people add specifically to make `pgrep` safe.

The agent's first public summary of this also got the count wrong: it said both in-tool cases were stopped. One was. The correction came from rereading the session log when the maintainer asked whether the kills had been staged. They had not.
