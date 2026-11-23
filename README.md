# agent-incidents

Real incidents caused by AI coding agents during day-to-day software work, written down as they happened.

Each entry records what the agent ran, what broke, how it was noticed, and how it was repaired. Details that identify machines or accounts are removed; commands and timings are otherwise taken from the session logs.

| Date | Incident | Blast radius | Time to repair |
|---|---|---|---|
| 2026-11-06 | [An agent ran `rm -rf -- /dev/null` as root](incidents/2026-11-06-rm-dev-null.md) | One shared Linux test machine | About 50 seconds |
| 2026-11-21 | [An agent's awk tree walk put PID 1 in a root `kill` list](incidents/2026-11-21-awk-kill-pid1.md) | One shared Linux container: every lane's working files in /tmp | Container back in about 1 minute; shared caches rebuilt separately |
| 2026-11-23 | [The same awk tree walk restarted the container twice more](incidents/2026-11-23-awk-kill-pid1-again.md) | The same container, twice in 41 minutes: every lane's /tmp | About 1 minute per restart; shared caches rebuilt by a boot job |
