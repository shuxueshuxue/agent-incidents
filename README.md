# agent-incidents

Real incidents caused by AI coding agents during day-to-day software work, written down as they happened.

Each entry records what the agent ran, what broke, how it was noticed, and how it was repaired. Details that identify machines or accounts are removed; commands and timings are otherwise taken from the session logs.

| Date | Incident | Blast radius | Time to repair |
|---|---|---|---|
| 2026-11-06 | [An agent ran `rm -rf -- /dev/null` as root](incidents/2026-11-06-rm-dev-null.md) | One shared Linux test machine | About 50 seconds |
