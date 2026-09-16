# Playerbots at scale

Running hundreds of random bots exposes bottlenecks that never appear with a handful. All figures
below were measured on a 12-core / 42 GB/s-memory host with 500–1500 bots.

## Database pool threading

Each pool opens `WorkerThreads` **async** connections and `SynchThreads` **synchronous** ones
(`worldserver.conf`, plus `PlayerbotsDatabase.*` in `playerbots.conf`). All default to 1.

- With `WorkerThreads = 1`, every bot login's character/inventory/spell/quest/aura queries serialize
  through a single connection, and a **player's** queries queue behind them. Symptom: a spell or pet
  summon that lands a minute after the cast, while the world tick itself is healthy (median ~1 ms).
- Raising `CharacterDatabase.WorkerThreads` to 4 took bot login from ~16/min to ~140/min.
- **But raising it above 1 deadlocks on MySQL's default isolation.** Character saves run
  `DELETE FROM character_aura WHERE guid = ?` (`CHAR_DEL_CHAR_AURA`), and `character_aura`'s primary
  key leads with `guid`, so that is a **range** delete. Under REPEATABLE-READ it takes gap locks, and
  two bots saving concurrently deadlock: `[1213] Deadlock found when trying to get lock`. The losing
  transaction's character save is **silently lost**.
- Fix the isolation rather than reverting the concurrency: start MySQL with
  `--transaction-isolation=READ-COMMITTED`. It does not gap-lock, so per-character delete+insert is
  safe in parallel. `WorkerThreads > 1` without it is a data-integrity bug, not a tuning choice.

Watch `SHOW STATUS LIKE 'Innodb_row_lock_waits'` when changing these — a climbing value is the
warning that precedes deadlocks.

MySQL's stock 128 MB `innodb_buffer_pool_size` is also smaller than the three databases
(~563 MB combined), so the working set cannot be cached at all.

## Bot login ramp

`RandomPlayerbotMgr::UpdateAIInternal` splits a per-cycle budget between updating online bots and
logging in new ones:

```cpp
uint32 updateBots = randomBotsPerInterval * onlineBotFocus / 100;  // focus is 25 while ramping, else 75
uint32 loginBots  = std::min(randomBotsPerInterval - updateBots, maxNewBots);
```

So the ramp slows as more bots come online — the budget shifts toward updates.

`botLoading` holds bots whose character load is in flight. Gating new logins on `botLoading.empty()`
makes the ramp **drain-then-refill**: nothing new is queued until the slowest load of the previous
batch finishes, leaving the character-DB worker idle in between. Prefer a bounded in-flight count so
loads pipeline.

## Scheduling the ramp away from players

`AiPlayerbot.DisabledWithoutRealPlayerLoginDelay` (non-zero) holds bots offline until **after** a
real player logs in — which guarantees the entire ramp lands on that player's session. On a host
where the ramp takes minutes, set it to `0` so bots populate at server start instead, and leave them
online.

## Keeping the player responsive

`BotActiveAlone` is the percentage of bots fully simulated when no player is near, and
`botActiveAloneSmartScale` scales that down as the world tick degrades (`...DiffLimitfloor` →
no reduction, `...DiffLimitCeiling` → all non-forced bots paused).

The force rules (`BotActiveAloneForceWhenInRadius`, `...InZone`, `...InGuild`) **always win over
SmartScale**, as do combat, dungeons/BGs, and grouping with a real player. That is the lever for
"busy where the player is, quiet elsewhere": keep `InRadius`, and drop `InZone` at high bot counts
since it force-wakes every bot in the zone.
