# Chapter 13 — Making Conversations Survive: Persistence

> **This chapter answers:**
> - How is a conversation saved between turns and resumed after you quit EvoScientist and come back the next day?
> - Why would the naive "snapshot everything each step" approach grow `sessions.db` to multiple gigabytes?
> - What is a *delta channel*, and why does it make pruning dangerous — dangerous enough to silently erase your history?
> - How does EvoScientist prune safely so that `/resume` still shows the full transcript?

On the master map from Chapter 2, this chapter zooms into the region labelled **Persistence (checkpointer, sessions.db)** — the small box at the bottom of the diagram that every surface quietly leans on. When you type a message in the CLI, the reply streams back, and you close your laptop, something has to remember that the conversation happened. That "something" is one SQLite file and one Python class, and the class is not what you'd expect: it is not agent logic, it is a piece of *disciplined database housekeeping*. The largest single source file in the whole project — `sessions.py`, 1868 lines — exists mostly to solve a problem that sounds trivial ("save the conversation") but hides a trap sharp enough to wipe a user's history without any error message at all. This chapter is the story of that trap and the guardrail built around it.

You already met the core idea in Chapter 3. There we said: LangGraph's **checkpointer** (the component that snapshots graph state after each super-step so a run can resume) writes a snapshot keyed by **thread_id** (the id grouping one conversation's chain of checkpoints), and that is how resume works. That was the intuition. This chapter is the depth: where the snapshots actually go, what happens when they pile up, and the one subtlety — the delta channel — that turns "just keep the recent ones" from an obvious optimization into a data-loss bug.

## Where the conversation lives: one file, three directories

Start with the plainest question: *where on disk is your conversation?*

The answer is a single SQLite database at `~/.evoscientist/sessions.db`. SQLite is an embedded relational database that lives in one ordinary file — no server process, no network port, just a file the library reads and writes directly. LangGraph ships a checkpointer backend called `AsyncSqliteSaver` that stores checkpoints into exactly such a file, using two tables: `checkpoints` (one row per snapshot) and `writes` (the incremental updates attached to each snapshot). EvoScientist points that backend at one shared file for every conversation you ever have.

The path is resolved in `get_db_path` (`EvoScientist/sessions.py:108`):

```python
def get_db_path() -> Path:
    from .paths import DATA_DIR

    db_dir = DATA_DIR
    db_dir.mkdir(parents=True, exist_ok=True)
    return Path(_to_short_path(str(db_dir))) / "sessions.db"
```

`DATA_DIR` comes from `paths.py`, and it is worth pausing on *why* the file lives where it does, because EvoScientist deliberately splits its on-disk footprint into three distinct kinds of location — a design decision that keeps upgrades and backups sane.

The first is the **workspace root** (`paths.py:26`): by default your current working directory, overridable with `EVOSCIENTIST_WORKSPACE_DIR`. This holds artifacts tied to *a particular project* — `runs/`, `skills/`, `media/`. Different projects, different workspaces.

The second is the **global data directory**, `~/.evoscientist/` (`paths.py:44`), overridable with `EVOSCIENTIST_DATA_DIR`. Its docstring draws the line explicitly (`paths.py:34-40`):

```python
    """Global application data directory (~/.evoscientist/ by default).

    This is the base for sessions.db, skills/, memories/, history — things
    that are NOT configuration but application state. Config files (config.yaml,
    mcp.yaml) continue to live in XDG_CONFIG_HOME.
    """
```

This is *application state* — data the program accumulates as it runs, shared across all your workspaces. `sessions.db` lives here. It is not tied to one project, and it is not something you hand-edit.

The third is the **configuration directory**, `~/.config/evoscientist/` (following the XDG_CONFIG_HOME convention that Unix tools use for user config). This holds files *you* write to steer the program — `config.yaml`, `mcp.yaml` — plus, as later chapters will show, the runtime bookkeeping for the `langgraph dev` subprocess.

The reason to separate state from configuration is that they have opposite lifecycles. You back up and hand-edit your config; you should never need to touch your session database. You might reset your config to defaults; you must never lose your sessions. Keeping them in different directories means a tool that clears one can never accidentally clear the other. (In fact `paths.py:72` carries a one-time migration, `migrate_legacy_sessions_db`, that copies an older `sessions.db` from the *config* directory into the new *data* directory — an earlier version of EvoScientist put it in the wrong place, and the fix has to move existing users' files without ever losing them. The migration is guarded by a `.migrated` marker file written only on success, so a failed copy is retried rather than skipped.)

So: one SQLite file, in the global data directory, holding every conversation as a chain of checkpoints. That is the whole storage model. Now the trouble begins.

## The problem: naive checkpointing explodes

Here is the mechanism you must hold in your head to understand everything that follows. LangGraph does not snapshot *the change* at each super-step — it snapshots *the entire state*. After super-step 1, it writes the whole conversation state. After super-step 2, it writes the whole state again — including everything from step 1, plus the new bit. After step 3, the whole thing once more. Each checkpoint is a full copy.

For a short exchange this is invisible. But a single research session in EvoScientist is not short. The agent reasons, calls tools, reads files, runs code, delegates to sub-agents — dozens, sometimes hundreds of super-steps, each one appending to a `messages` list that only grows. If every super-step re-serializes the full, growing state, the storage cost is not linear in the length of the conversation; it is *quadratic*. A conversation of N steps stores state of size 1 + 2 + 3 + … + N. And the messages themselves are not small: a single model turn can carry a multi-kilobyte reasoning trace, a tool result can be a whole file's contents.

The module docstring at the very top of `sessions.py:8-15` states the consequence plainly:

```python
Per-step pruning:
    LangGraph's checkpointer writes a full state snapshot per super-step,
    causing unbounded growth (multi-GB sessions.db). EvoScientist never
    reads historical checkpoints — resume always reads the latest, HITL
    interrupts attach pending writes to the just-written row. So
    ``get_checkpointer()`` yields a ``PruningCheckpointer`` that prunes
    older rows for the same ``(thread_id, checkpoint_ns)`` after every
    ``aput()``. The first-run migration sweep cleans up legacy bloat.
```

"Multi-GB" is not hyperbole. A constant elsewhere in the file (`sessions.py:1106`) mentions a "2.6 GB pathology" — a real database that had grown to 2.6 gigabytes of mostly-redundant snapshots. That is the disease.

Now read the cure in the same docstring, because it hinges on one crucial observation about how EvoScientist actually *uses* checkpoints: **it never reads historical ones.** When you `/resume`, EvoScientist wants exactly one thing — the *latest* state of the conversation, so it can continue. It never asks "what did the state look like 40 steps ago?" LangGraph's checkpointer supports time-travel to any past checkpoint; EvoScientist simply doesn't use that capability. And if you never read the old snapshots, you don't need to keep them.

That is the whole idea, and it is worth stating as a design principle before we see the mechanism, because the mechanism only makes sense in its light:

> **Principle.** Persist only what you will read back. EvoScientist reads back only the latest state of each conversation, so it keeps only a bounded window of recent checkpoints and deletes the rest as they age out.

The component that enforces this is the **PruningCheckpointer** (introduced here): EvoScientist's checkpointer that prunes old rows so `sessions.db` stays bounded. Every time LangGraph writes a checkpoint, the PruningCheckpointer immediately turns around and deletes the checkpoints that have now fallen outside a fixed-size retention window. The database stops growing; it reaches a steady state where each conversation keeps at most a bounded window of its most recent snapshots. The window is deliberately set well above any realistic conversation length (the default keeps up to a thousand — `config/settings.py:441-445` notes a normal conversation runs ~180–450 turns), so in ordinary use the pruning almost never fires against a live conversation; it exists to stop *unbounded* accumulation and to reclaim legacy or runaway bloat.

## A checkpointer that subclasses, and prunes after each write

Before the pruning logic, one small but instructive design detail: the way PruningCheckpointer attaches itself to LangGraph.

The obvious way to add "prune after each write" behavior would be the *wrapper* pattern — hold a reference to a real `AsyncSqliteSaver`, forward every call to it, and slip a prune step in after the write. EvoScientist deliberately does *not* do this. It uses inheritance instead (`sessions.py:155-168`):

```python
class PruningCheckpointer(AsyncSqliteSaver):
    """``AsyncSqliteSaver`` that prunes stale checkpoints after every ``aput()``.

    ...

    Inherits from ``AsyncSqliteSaver`` (rather than wrapping it) so
    LangGraph's ``compile()`` ``isinstance(x, BaseCheckpointSaver)`` check
    succeeds. All other behavior — ``aget_tuple``, ``alist``,
    ``aput_writes``, ``adelete_thread``, ``setup``, the async context
    manager protocol, the connection lock — is inherited unchanged.
```

The reason is a type check inside LangGraph itself. When LangGraph compiles a graph with a checkpointer, it verifies `isinstance(checkpointer, BaseCheckpointSaver)`. A wrapper object is *not* an instance of that class — it merely delegates to one — so a wrapper would fail the check. By subclassing `AsyncSqliteSaver` (which is itself a `BaseCheckpointSaver`), PruningCheckpointer *is* a genuine checkpointer as far as LangGraph is concerned, and it inherits every method — reads, deletes, connection locking, the async-context-manager protocol — for free. It overrides exactly one method (`aput`, the write) and adds the pruning. This is the smallest possible surface area: change what you must, inherit everything else, and stay a first-class citizen of the framework's type system.

`aput` is LangGraph's "put a checkpoint" method — the `a` prefix marks it `async`. Here is the override (`sessions.py:211-240`), which does the write, then the prune, both under a lock:

```python
    async def aput(
        self,
        config: Any,
        checkpoint: Any,
        metadata: Any,
        new_versions: Any,
    ) -> Any:
        async with self._aput_lock:
            result = await super().aput(config, checkpoint, metadata, new_versions)
            if self._keep_per_ns <= 0:
                return result
            try:
                thread_id = config["configurable"]["thread_id"]
                checkpoint_ns = config["configurable"].get("checkpoint_ns", "") or ""
                await self._prune_after_put(str(thread_id), str(checkpoint_ns))
            except Exception as exc:  # pragma: no cover - defensive
                _logger.warning("checkpoint pruning failed: %s", exc, exc_info=True)
            return result
```

Three details in these few lines encode hard-won discipline. First, `super().aput(...)` does the *real* write — the durable snapshot lands before any pruning is even considered, so the just-completed super-step is always saved. Second, the whole pair is wrapped in `self._aput_lock`, an `asyncio.Lock`. The docstring at `sessions.py:187-193` explains why: without it, a concurrent `aput()` on a *different* conversation could interleave between the write and the prune and squeeze this caller's just-written row out of its own retention window. The lock makes "write then prune" atomic as a pair, guaranteeing the invariant *the just-written row is always kept*. Third — and this is the temperament of the whole file — pruning is wrapped in a broad `try/except` that logs at WARNING and swallows the error. Pruning is *housekeeping*; it must never fail the agent step. If a transient SQLite hiccup makes a delete fail, the conversation still proceeds and the database is merely a little larger than ideal until the next write cleans up. Correctness of the conversation always outranks tidiness of the disk.

Notice the `checkpoint_ns` (checkpoint namespace) appearing alongside `thread_id`. A namespace is LangGraph's way of separating parallel sub-graphs within one thread — the main conversation lives in the empty namespace `""`, and nested runs get their own. Pruning operates per `(thread_id, checkpoint_ns)` pair, so each namespace keeps its own window. For the main conversation, the namespace is always `""`.

The retention window itself — how many recent checkpoints to keep — comes from configuration. `_resolve_keep_per_ns` (`sessions.py:454`) reads `EvoScientistConfig.checkpoint_keep_per_thread`, which defaults to `1000` (`config/settings.py:445`), and there is a matching fallback constant `_DEFAULT_KEEP_PER_NS = 1000` (`sessions.py:152`) for the paths where config can't be resolved (tests, early init). Setting it to `0` or below disables pruning entirely — an escape hatch for debugging. A thousand recent snapshots per conversation is generous; it caps the disk cost without ever pruning anything a normal user could notice.

So far the picture is simple: after each write, delete everything but the newest thousand rows for this conversation. If that were the whole story, this would be a short chapter. It is not, because of what LangGraph stores in those rows.

## The trap: delta channels

Here is where the naive design bites. To understand it we need to look at *how* the `messages` list is actually stored — because in modern LangGraph, it is not stored the way you'd assume.

Think about the quadratic-growth problem from LangGraph's own side. LangGraph's authors saw it too, and their fix (as of LangGraph 1.2) is a storage optimization called a **delta channel**: LangGraph's message storage that keeps periodic full snapshots plus incremental writes, replayed on read. A *channel* is one named slot of state — `messages` is a channel. Instead of storing the entire `messages` list in *every* checkpoint, a delta channel stores the full list only *occasionally* — call these the **snapshot** checkpoints — and in between it stores only the *new* messages added at each step as small **writes** in the `writes` table.

An analogy: a delta channel is like a video codec that stores an occasional full "keyframe" and, in between, only the pixels that changed. To display any given frame, the player finds the most recent keyframe before it and replays the changes forward. To reconstruct the `messages` list at any checkpoint, LangGraph does the same: it walks *backward* along the parent chain (each checkpoint records its `parent_checkpoint_id`) until it hits the nearest checkpoint whose `messages` channel holds a full snapshot, then replays all the incremental writes forward from there.

Here is the structure, reading right-to-left as the chain grows (arrows point from child to parent):

```
  ck_100        ck_99        ...    ck_51        ck_50 (SNAPSHOT)   ck_49 ...
 [+msg]  --->  [+msg]  ---> ...   [+msg]  --->  [full messages] --> ...
   ^                                                  ^
 latest                                     nearest snapshot ancestor
   |___________________ replay writes forward ________|
```

To reconstruct the state at `ck_100`, LangGraph starts at `ck_50` (the full snapshot), then applies the incremental writes from `ck_51`, `ck_52`, …, up to `ck_100`. The full message list at `ck_100` exists *nowhere* as a single stored object — it is computed on read.

Now re-examine naive pruning against this picture. "Keep the newest thousand checkpoints, delete the rest." Suppose your conversation has grown past the window and the newest thousand are `ck_101` … `ck_1100`. The snapshot they all depend on is `ck_50` — far outside the window. Naive pruning deletes `ck_50`.

What happens on your next `/resume`? LangGraph walks back from the latest checkpoint looking for a full snapshot to replay from. It never finds one — `ck_50` is gone, and the surviving checkpoints hold only incremental writes. With no seed to start from, reconstruction begins from an empty list. **Your `/resume` silently shows an empty conversation.** No error, no warning — LangGraph reconstructs *something*, it just reconstructs *nothing*, because the keyframe it needed was pruned away.

This is the trap, and its danger is precisely that it is *silent*. The code even documents that upstream LangGraph's own generic `prune` helper suffers the same failure mode. `sessions.py:250-257` names it:

```python
    async def _prune_after_put(self, thread_id: str, checkpoint_ns: str) -> None:
        """Prune old checkpoints with DeltaChannel awareness.

        Naively keeping the N most-recent rows can sever the
        ``_DeltaSnapshot`` chain that ``messages`` reconstruction
        depends on — the surviving "latest" checkpoint is rarely a
        snapshot point itself, so delta channels silently reconstruct
        as empty (upstream ``BaseCheckpointSaver.prune`` spells out the
        same failure mode).
```

The lesson generalizes far beyond this codebase, and it is the intellectual core of the chapter: **when you compact a linked structure, you cannot delete purely by recency — you must preserve whatever the surviving nodes still depend on to be reconstructed.** A delta channel is a linked chain where recent nodes depend on an older seed. Prune the seed and the recent nodes become unreadable. The same shape appears anywhere incremental state is stored against periodic baselines: log-structured stores, incremental backups, event-sourced systems with snapshots. EvoScientist's fix for the `messages` channel is a concrete embodiment of a rule that applies to all of them.

## Pruning safely: walk to the snapshot ancestor

The fix follows directly from the problem statement. Keep the newest window of checkpoints as before — but before deleting the rest, walk back from the *oldest* checkpoint you're keeping until you find the snapshot it depends on, and protect that snapshot (and everything on the path to it) from deletion too.

The orchestration lives in `_prune_after_put` (`sessions.py:276-296`):

```python
        keep = self._keep_per_ns
        agent = AGENT_NAME
        async with self.lock:
            anchor_ids = await self._fetch_recent_checkpoint_ids(
                thread_id, checkpoint_ns, agent, keep
            )
            if len(anchor_ids) < keep:
                return  # nothing to prune yet

            extra_preserve = await self._walk_to_snapshot_ancestor(
                thread_id, checkpoint_ns, anchor_ids[-1]
            )
            kept = set(anchor_ids) | extra_preserve

            await self._delete_outside(thread_id, checkpoint_ns, agent, kept)
            await self.conn.commit()
```

Read it as three moves. First, `_fetch_recent_checkpoint_ids` grabs the `keep` most-recent checkpoint ids for this conversation, newest first — these are the **anchors**, the window we always keep. If there aren't even `keep` of them yet, there is nothing to prune and we return early. Second, `_walk_to_snapshot_ancestor` starts from `anchor_ids[-1]` — the *oldest* anchor — and walks its parent chain to find the snapshot seed it depends on, returning the extra ids to preserve. Because the anchors form a contiguous run of the most recent checkpoints, the snapshot that the *oldest* anchor needs is also the one all the newer anchors need; a single walk from the oldest covers the whole window. Third, `_delete_outside` deletes everything *not* in the union of anchors and preserved ancestors, and commits.

The walk itself (`sessions.py:318-358`) is the safety-critical heart of the file. Read it slowly:

```python
    async def _walk_to_snapshot_ancestor(
        self,
        thread_id: str,
        checkpoint_ns: str,
        oldest_anchor_id: str,
    ) -> set[str]:
        """Walk parent chain until hitting a ``messages`` seed.

        Returns the set of ancestor ids to preserve (inclusive of the
        snapshot ancestor). On chain-break or deserialization failure,
        returns what was visited so far — the safe side is over-preserve.
        """
        extra: set[str] = set()
        cursor = await self._fetch_parent_checkpoint_id(
            thread_id, checkpoint_ns, oldest_anchor_id
        )
        steps = 0
        while cursor is not None and steps < self._MAX_SNAPSHOT_WALK_STEPS:
            steps += 1
            blob = await self._fetch_checkpoint_blob(thread_id, checkpoint_ns, cursor)
            if blob is None:
                break  # chain broken (legacy DB); preserve what we have
            extra.add(cursor)
            try:
                ck = self.serde.loads_typed(blob)
            except Exception as exc:
                _logger.warning(...)
                break  # safe-side: preserve everything visited so far
            cv = ck.get("channel_values") or {}
            if _unwrap_messages_seed(cv.get("messages")) is not None:
                break  # found seed; this ancestor anchors reconstruction
            cursor = await self._fetch_parent_checkpoint_id(
                thread_id, checkpoint_ns, cursor
            )
        return extra
```

The loop starts one step *above* the oldest anchor — at its parent — because the anchors themselves are already in the keep set. Each iteration reads one checkpoint's stored blob, deserializes it (`self.serde.loads_typed`, the checkpointer's serializer), inspects its `messages` channel, and asks: *is this a snapshot seed?* If yes, we have found the keyframe; the walk stops, and this ancestor is preserved because it was added to `extra` before the check. If no, this checkpoint holds only incremental writes, so we mark it preserved anyway (a full walk of the path must survive) and step to *its* parent.

Every exit path over-preserves rather than under-preserves — and that asymmetry is deliberate, because the two errors are not equal. Over-preserving keeps a few extra rows the pruner *could* have deleted: harmless, self-correcting on the next write. Under-preserving deletes a seed and silently empties someone's history: catastrophic and invisible. So every branch chooses the safe side. If the chain breaks — `blob is None`, meaning a parent id points at a row that doesn't exist, as happens in legacy databases — the walk stops and keeps what it has seen. If deserialization throws, same thing: log and stop, preserving the visited set. And a hard ceiling, `_MAX_SNAPSHOT_WALK_STEPS = 10000` (`sessions.py:247`), caps the walk so a pathological cycle or a corrupt parent pointer can never spin forever; the comment notes this is a generous ceiling well above LangGraph's default snapshot frequency, chosen to catch malformed data without limiting normal operation.

The seed-detection helper `_unwrap_messages_seed` (`sessions.py:785-802`) is small but carries a genuine gotcha worth quoting, because it is the kind of bug that only bites once a library upgrades under you:

```python
def _unwrap_messages_seed(value: object) -> list | None:
    """Coerce a ``channel_values["messages"]`` snapshot seed into a plain list.

    LangGraph 1.2 stores snapshot blobs as ``_DeltaSnapshot(value=[...])``
    (a ``NamedTuple``, NOT a list subclass), so a bare ``isinstance(v, list)``
    check silently ignores the seed and reconstruction starts from whatever
    writes survived pruning. Pre-migration / non-DeltaChannel checkpoints
    still store a plain list. Returns ``None`` when no usable seed is
    present (caller leaves accumulated state untouched).
    """
    if value is None:
        return None
    if isinstance(value, list):
        return list(value)
    inner = getattr(value, "value", None)
    if isinstance(inner, list):
        return list(inner)
    return None
```

Here is the subtlety in one sentence: LangGraph 1.2 wraps a snapshot in a `_DeltaSnapshot` named tuple, which is *not* a list, so the intuitive check `isinstance(value, list)` would look at a real snapshot and conclude "not a seed." A pruner using that check would walk right past the keyframe and delete it — reintroducing the exact bug it was meant to prevent. The helper therefore handles both shapes: a plain list (older, non-delta checkpoints) *and* a `_DeltaSnapshot` whose `.value` attribute is the list (LangGraph 1.2). The code deliberately depends on a pre-stable, private LangGraph shape, and it knows it — which is why detection is quarantined in this one small, heavily-commented function that is easy to update when the library moves again.

The actual deletion in `_delete_outside` (`sessions.py:388-446`) carries one more piece of ordering discipline: it deletes from the `writes` table *before* the `checkpoints` table. If it dropped checkpoints first, the surviving writes' `checkpoint_id` references would become orphans pointing at rows that no longer exist. Writes-first preserves referential ordering. (It also guards for legacy databases that predate the `writes` table entirely, skipping that step if the table is absent — the same defensiveness runs throughout the file.)

## Reading it back: how `/resume` reconstructs history

Pruning keeps the seed alive; reconstruction is what actually *uses* it. When you `/resume`, EvoScientist has to turn the surviving rows back into the message list you'll see. That path is `_load_checkpoint_messages` (`sessions.py:621`), and it is the mirror image of the pruner: where the pruner *walks back to* the snapshot ancestor to protect it, the reader *walks back to* the snapshot ancestor to replay from it.

The reader delegates the walk to LangGraph's own `aget_delta_channel_history`, which "finds the nearest ancestor whose `channel_values["messages"]` carries a seed … and returns that plus every on-path pending write oldest→newest" (`sessions.py:627-635`). Then it replays those writes forward with a reducer to rebuild the list — dedup by message id, honoring `RemoveMessage` tombstones and the `REMOVE_ALL_MESSAGES` reset. One telling detail: EvoScientist keeps its *own inline copy* of that reducer, `_reduce_messages_delta` (`sessions.py:547`), rather than importing it, because the upstream reducer lives in a private, pre-stable module. The docstring is candid: "We copy the implementation here so a future upstream rename or semantic shift doesn't silently break thread reconstruction." This is the same instinct as `_unwrap_messages_seed` — reconstruction is too important to leave hostage to a library's private internals, so the fragile coupling is pulled into the open where it can be maintained.

Two more precautions round out the reader. Before the walk, it *pins* the exact latest checkpoint id of the EvoScientist-owned head into the config (`sessions.py:658-676`), rather than trusting `aget_tuple` to pick "the latest" on its own — because this one database is shared with other graphs, and a third-party row with a higher id could otherwise leak its transcript into your `/resume`. And when reconstruction comes back *empty*, it calls `_log_orphan_warning_if_pruned` (`sessions.py:745`) to check whether the emptiness was caused by a severed chain — turning the previously-silent failure into a diagnosable warning. That is the guardrail acknowledging its own worst case: if pruning ever does sever a chain despite all the care, the system will at least *say so* instead of quietly showing you nothing.

## Cleaning up the legacy bloat, and reclaiming disk

Two housekeeping mechanisms finish the persistence story. Both exist because pruning going forward doesn't help a database that already grew to gigabytes before the pruner existed.

The first is a **one-time migration sweep**. `get_checkpointer` (`sessions.py:464`) — the context manager the CLI enters at `cli/interactive.py:708` to get its checkpointer — runs a bulk prune *before* yielding the saver, but only when needed. "When needed" is decided by `_needs_migration` (`sessions.py:1196`): the sweep runs only if the database exists, is larger than `_MIGRATION_THRESHOLD_BYTES` (100 MB, `sessions.py:1107`), and its `PRAGMA user_version` is below the current migration version. `user_version` is a 32-bit integer slot in the SQLite file header — a place to stamp "I have already run migration N." On success the sweep bumps it (`sessions.py:1280`) so it never reruns; on failure it leaves the version untouched so the next launch retries. During the sweep the user sees a Rich spinner — "Compacting sessions DB…" with a size estimate and progress — because on a 2.6 GB file this takes real time, and the code yields to the event loop between thread-namespace pairs so the agent stays responsive. This is the "first-run migration sweep cleans up legacy bloat" promised in the module docstring: a normally-pruned database never crosses 100 MB, so the sweep fires at most once, for users upgrading from a pre-pruning version.

The second is `VACUUM`. Deleting rows from SQLite frees the *logical* space but does not shrink the *file* — the freed pages sit inside the file marked reusable. `VACUUM` rewrites the database compactly and hands the disk space back to the operating system. But `VACUUM` needs an exclusive lock on the database, which it cannot get while the agent's long-lived connection is open. So EvoScientist defers it to `atexit` (`sessions.py:1293`, `_schedule_vacuum_atexit`): the compaction runs *at process exit*, after the saver's connection has closed, using the stdlib `sqlite3` module (since `atexit` handlers can't `await` the async driver). It is registered exactly once per process and captures the resolved path up front so that test monkey-patching of `get_db_path` can never cause an accidental `VACUUM` on production data. Best-effort, deferred, path-frozen: three small defenses on one cleanup step.

## The WebUI's checkpointer, and why it exists

> **背景 → 经过 → 代价 → 机制化 (Background → What happened → Cost → Mechanized)**
>
> **背景 (Background).** EvoScientist's WebUI and its background work don't run in the CLI process — they run inside a separate `langgraph dev` subprocess (Chapter 14 owns that story). That server needs a checkpointer of its own, and out of the box LangGraph hands it a default `InMemorySaver`: a checkpointer that keeps state in memory and flushes it to a pickle file on a timer.
>
> **经过 (What happened).** Documented as issue #277 (`sessions.py:1842-1844`), the `InMemorySaver` has two failure modes. Its flush window is about 10 seconds, so if the process is `SIGKILL`ed — a hard, no-cleanup kill — the checkpoints written in the last window are simply gone. Worse, it persists via *pickle*, Python's object-serialization format, which is version-sensitive: a library upgrade that changes an object's shape can make the old pickle file unreadable, and a pickle-incompatible upgrade "wipe[s] the whole store." Users lost session history on restart.
>
> **代价 (Cost).** Silent loss of recent conversation on any crash, and total loss of the WebUI's session store across certain upgrades — the same class of "history silently vanishes" failure the CLI's own pruner works so hard to avoid, arriving through a different door.
>
> **机制化 (Mechanized).** EvoScientist replaces the default with its own SQLite-backed checkpointer for the subprocess. `create_checkpointer_for_langgraph_api` (`sessions.py:1836`) is registered as the `checkpointer.path` target in the subprocess's `langgraph.json` manifest, so every `langgraph dev` launch uses the *same* `sessions.db` file the CLI uses. Now "every `aput()` commits a WAL transaction and a bad row only loses that row" — durability per super-step, and no pickle to break on upgrade. It yields an `_ApiPruningCheckpointer` (`sessions.py:1448`), a subclass that additionally stamps each row with `workspace_dir`, `updated_at`, and — for main-graph rows — `agent_name`, so the WebUI's conversations become first-class CLI sessions that show up in `/resume` and participate in the same pruning. The rule that came out of the incident: *never let a background server hold session state in a volatile, upgrade-fragile store — back it with the same durable file the rest of the system trusts.*

## One file, many graphs: ownership and the thread model

The last thread to pull is why this file is so careful about *whose* rows it touches. `sessions.db` is shared. The CLI writes to it, the WebUI subprocess writes to it, EvoScientist's background memory-worker graphs write to it, and in principle any third-party tool built on the same LangGraph SQLite backend could write to it too. Every operation in `sessions.py` that lists, deletes, or prunes must therefore ask: *is this row mine?*

The answer is a metadata filter. `MAIN_THREAD_FILTER_SQL` (`sessions.py:73-78`) restricts every listing and deletion to rows whose metadata carries `agent_name == "EvoScientist"` (and whose `graph_id` is either the main graph or absent). The pruner uses the same predicate, and there is a sharp subtlety in *how* SQLite's `json_extract` behaves on rows without that key: it returns NULL, and `NULL = 'EvoScientist'` is false, so a row lacking an `agent_name` is never matched — never listed, never pruned. The docstring at `sessions.py:267-271` states the intent plainly: "those rows belong to third-party LangGraph users and must never be pruned by us." The pruner is a good neighbor by construction; it can only ever delete EvoScientist's own history.

Thread identity carries its own small design choice. `generate_thread_id` (`sessions.py:131-140`) returns a *full* UUID rather than the short 8-character hex id an earlier version used:

```python
def generate_thread_id() -> str:
    """Generate a full-UUID thread ID.

    UUID format (not the legacy 8-char hex) so CLI threads are addressable
    by langgraph-api — its thread endpoints reject non-UUID ids — which is
    what lets the WebUI list and resume CLI sessions. UIs display the
    first 8 chars; ``/resume`` prefix matching is unaffected. Pre-existing
    8-char hex threads keep working in the CLI but stay CLI-only.
    """
    return str(uuid.uuid4())
```

The reason is interoperability: LangGraph's HTTP API rejects non-UUID thread ids, so a CLI thread with a UUID can be listed and resumed *from the WebUI*, while a legacy 8-hex thread stays CLI-only. For humans, `short_thread_id` (`sessions.py:122`) shows just the first 8 characters (git-style), and every lookup command accepts prefixes, so the short form is still a usable handle even though the stored id is a full UUID. Around these ids, `sessions.py` provides ordinary thread CRUD — `list_threads` (`sessions.py:877`) groups checkpoints by `thread_id`, taking the max `updated_at` and a first-human-message preview for the session picker; `delete_thread` (`sessions.py:1008`) removes a conversation's rows, again writes-before-checkpoints. These are the operations the CLI's `/resume` and `/delete` commands sit on, and the surfaces that call them are Chapter 15's subject.

## Takeaways（要点）

- **The storage model is one SQLite file of full snapshots.** A conversation is a **thread**; its state is a chain of checkpoints in `~/.evoscientist/sessions.db`. `sessions.db` lives in the *global data directory*, kept deliberately separate from the *workspace* (per-project artifacts) and the *config directory* (user-editable settings) so their opposite lifecycles never collide.
- **Naive checkpointing grows quadratically.** LangGraph snapshots the *entire* state after every super-step, and the `messages` list only grows, so an unpruned database reaches multiple gigabytes (a real 2.6 GB case is documented in the code). EvoScientist never reads historical checkpoints — resume reads only the latest — so it keeps a bounded window (default 1000) and deletes the rest.
- **PruningCheckpointer subclasses `AsyncSqliteSaver` and prunes after each `aput`.** Subclassing (not wrapping) keeps it a genuine `BaseCheckpointSaver` so LangGraph's `isinstance` check passes. The write-then-prune pair runs under a lock so the just-written row is always kept, and pruning failures are logged-and-swallowed so housekeeping never breaks the agent step.
- **Delta channels make recency-only pruning a silent data-loss bug.** LangGraph 1.2's `messages` channel stores periodic full snapshots plus incremental writes; reconstruction walks back to the nearest snapshot ancestor. Delete that ancestor and `/resume` silently reconstructs *empty history* — no error at all.
- **The fix is to walk to the snapshot ancestor and preserve it.** `_walk_to_snapshot_ancestor` climbs the parent chain from the oldest kept checkpoint until it finds the seed, preserving the whole path. Every failure branch *over*-preserves, because keeping too much is harmless and keeping too little is catastrophic and invisible. The general lesson: *compacting a linked structure requires preserving what surviving nodes depend on, not just the recent nodes.*
- **Cleanup is layered and defensive.** A one-time migration sweep (gated by `PRAGMA user_version`, triggered above 100 MB) removes legacy bloat; `VACUUM` is deferred to `atexit` to reclaim disk once the connection closes. A shared database is respected via an `agent_name` metadata filter so EvoScientist only ever prunes its own rows, and the WebUI subprocess is backed by this same durable file instead of the crash-and-upgrade-fragile default `InMemorySaver` (issue #277).

## Sources

*When this book and the code disagree, the code wins.* This chapter is grounded in:

| Topic | File(s) |
|---|---|
| The persistence problem + pruning rationale | `EvoScientist/sessions.py` (module docstring, `:1-27`) |
| On-disk layout: workspace / data / config dirs | `EvoScientist/paths.py` (`:26`, `:33-44`, `:72` migration) |
| PruningCheckpointer: subclass, `aput`, lock | `EvoScientist/sessions.py:155-240` |
| Delta-channel-aware pruning + the ancestor walk | `EvoScientist/sessions.py:249-358`, `:785-802` |
| Safe deletion ordering (writes before checkpoints) | `EvoScientist/sessions.py:388-446` |
| Resume / reconstruction | `EvoScientist/sessions.py:547-618`, `:621-749` |
| Migration sweep + `user_version` gating + `VACUUM` at exit | `EvoScientist/sessions.py:464-533`, `:1095-1334` |
| Thread ids, ownership filter, thread CRUD | `EvoScientist/sessions.py:73-78`, `:122-140`, `:877-1032` |
| WebUI / `langgraph dev` checkpointer (issue #277) | `EvoScientist/sessions.py:1836-1868`, `:1448` |
| Retention default | `EvoScientist/config/settings.py:445` |
| Where the CLI enters the checkpointer | `EvoScientist/cli/interactive.py:708` |
