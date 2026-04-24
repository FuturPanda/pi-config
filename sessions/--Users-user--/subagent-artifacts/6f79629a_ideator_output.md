Let me work through this properly, because several of these questions are entangled in non-obvious ways. The framing of "MongoDB + Redis caching" undersells the actual hard problem here.

---

**The real problem isn't caching. It's that `source_path` makes loading stateful.**

Most caching problems are pure read-path optimizations. This one has a write side-effect baked into the read: you must materialize files to disk before you can return a valid `Skill`. That changes everything. Questions 1–3 are actually downstream of this constraint, so let's start there.

---

**Question 5 first: source_path materialization**

This is the load-bearing question. `agent_skills.py` reads from local disk unconditionally. So regardless of whether you hit Redis or MongoDB, if the files aren't on disk, the Skill is unusable. Your loader has to materialize before returning.

The naive approach is `base_dir/{skill_name}/scripts/` and `base_dir/{skill_name}/references/`. Fine for a single instance. Falls apart under concurrency — two pods racing to write, or Pod A reading a partially-written file that Pod B is in the middle of creating.

The clean solution is **content-addressed paths**: `base_dir/{skill_name}-{content_hash}/`. You hash the combined content of all scripts and references at document-read time. Now:

- If two instances materialize simultaneously, they're writing identical bytes to the same path. The final state is correct regardless of ordering. The "race" becomes harmless.
- If a skill is updated in MongoDB, the hash changes, so it gets a *new* path. The old path stays on disk (you need a cleanup sweep), but there's no window where a half-migrated state is visible.
- Checking whether materialization is needed becomes a cheap `os.path.exists(derived_path)` — no need to re-hash or re-read.

One more thing: always write atomically. Write to a `.tmp` file, then `os.rename()`. Rename is atomic on POSIX systems. This closes the partial-read window entirely.

---

**Question 1: MongoDB schema — inline vs. references**

Given the above, inline content in the MongoDB document is strongly preferable. Store the actual text of each script and reference file directly in the skill document. Yes, this inflates document size, but skill files are almost certainly under MongoDB's 16MB document limit by orders of magnitude. In exchange you get:

- A single round-trip reads everything needed for materialization
- The document is self-contained — no secondary fetches from GridFS or S3
- The cache key for Redis can cover the *entire* skill including content

If you use GridFS or S3 pointers, you're trading document simplicity for multiple round-trips, which poisons your Redis cache hit path. A "Redis hit" now still requires fetching file content from somewhere else. That's not a cache hit in any meaningful sense.

Schema shape: one document per skill, with `scripts` and `references` as arrays of `{filename: str, content: str}` objects rather than bare filename lists.

---

**Question 2 & 7: What goes in Redis — raw document or materialized Skill?**

Cache the **raw MongoDB document**, not the `Skill` dataclass with `source_path` filled in.

If you cache the materialized Skill, `source_path` is baked in as a string. In a container environment, `/tmp/skills/...` paths are per-instance. A different pod reads that cache entry and gets a path that doesn't exist on its own disk. You've just created a ghost reference.

With the raw document in Redis, every instance that gets a cache hit still does the cheap disk-existence check and materializes if needed. That's slightly more work on a cache hit, but it's O(1) filesystem operations, not a network round-trip. And it keeps Redis machine-agnostic, which is the whole point of shared infrastructure.

---

**Question 3: Cache key strategy**

Don't cache `skills:all` as a single blob. Tempting because `load()` returns `List[Skill]`, but it means any single skill update busts the entire cache for all consumers simultaneously — thundering herd straight to MongoDB.

Instead: one key per skill, `skill:{name}`, storing the serialized MongoDB document. Then a separate index key, `skills:index`, storing the list of skill names. `load()` fetches the index, then pipelining-fetches all `skill:{name}` keys in one round-trip batch. Much cheaper than N serial fetches.

The index key is the weak point — it's a fan-out invalidation target. When you add/remove a skill, you need to invalidate or update `skills:index`. More on that under invalidation.

---

**Question 4 & 6: TTL vs. event-driven invalidation**

TTL alone is almost certainly wrong for agent skills in production. Here's why: stale skills don't just return old data — they execute wrong code. The blast radius of "skill was updated but cache still has old version" is agent misbehavior, not just a stale webpage. The staleness window matters a lot.

The right model is **TTL as a safety net, change streams as the primary invalidation mechanism**.

MongoDB change streams let you watch a collection for updates and fire a Redis DEL (or PUBLISH to a channel) immediately on write. This gets your invalidation latency from TTL-seconds down to sub-second. The TTL (set conservatively, say 1 hour) exists to handle cases where the change stream consumer misses an event — restarts, network hiccups, etc.

The practical question is who runs the change stream consumer. It shouldn't be the SkillLoader itself — that conflates two different lifecycles (request-time loading vs. background event processing). It's a separate process or sidecar. If that feels heavyweight, then you're back to TTL-only, but you should be honest about the trade-off you're accepting.

---

**Question 8: Cache miss behavior**

Redis miss → MongoDB hit: load document, materialize to disk, `SET` in Redis with TTL, return Skill. Standard.

Redis miss → MongoDB miss: the skill doesn't exist. `load()` should return an empty list, not raise — the contract doesn't imply skills always exist. But you should distinguish between "skill not found" and "MongoDB is down." The former is normal; the latter should propagate as an error, not silently return empty.

One subtle thing: after a Redis miss and successful MongoDB load, use `SET NX EX` (set if not exists, with expiry) rather than a plain `SET`. If two instances raced to MongoDB and are both now trying to populate Redis, `SET NX` means the second write is a no-op instead of a potential overwrite with slightly stale data. Low-stakes in practice, but it's the correct idiom.

---

**Question 9: Race conditions**

Content-addressed paths already handle the disk race (covered above). The Redis write race with `SET NX` handles the cache population race. The remaining interesting race: what if MongoDB is updated *while* two instances are both in a cold-start Redis miss?

Instance A reads version 1. MongoDB gets updated to version 2. Instance B reads version 2. A writes `skill:foo` with version 1 content. B writes `skill:foo` with version 2 content (or vice versa depending on timing). The cache ends up with either version, and you can't tell which.

The clean fix is **versioned cache keys**: `skill:{name}:{version_hash}`. Every document in MongoDB has a content hash or monotonic version. The cache key includes the version. Now there's no ambiguity — stale keys just don't get hit. This does mean you need the `skills:index` to store `{name: version_hash}` pairs, not just names, and invalidation is a key deletion rather than an overwrite. More moving parts, but eliminates the version-confusion race entirely.

---

**Question 10: Two-layer cache — Redis + in-process dict**

Yes, but only if you're careful about what you're caching locally.

Don't cache the raw document locally — you'd just be duplicating Redis with worse invalidation. What you *can* cache locally is: "I already materialized skill X with hash H, files are on disk." This is essentially a `Set[str]` of content-addressed paths you've already verified exist. Extremely cheap, no staleness risk (content-addressed paths are immutable by definition), and it avoids the `os.path.exists()` call on every `load()` invocation within the same process.

For the actual skill content, a short-lived local cache (30–60 seconds) makes sense if `load()` is called very frequently — say, on every agent invocation. The invalidation story is: local cache TTL is shorter than Redis TTL, which is shorter than the change stream invalidation latency. You're stacking consistency windows, which is fine as long as you document the maximum staleness: local TTL + stream latency in the worst case.

---

**The cleanest overall architecture**

The SkillLoader is responsible for: read from Redis → fallback to MongoDB → materialize to disk → populate Redis → return Skills. That's it. It should not run change streams, not manage TTL policy beyond passing it to Redis, not do cleanup of old materialized directories.

The change stream consumer is a separate process. Disk cleanup (old content-addressed directories) is a cron job or startup sweep. These are different operational concerns with different lifetimes.

The single most important design choice is content-addressed paths. It makes materialization idempotent, makes concurrent instances safe without locking, and makes cache coherence tractable. Everything else follows naturally from it.

One thing I'd push back on in the framing: calling this a "two-layer cache" understates what Redis is doing. Redis isn't just caching MongoDB query results — it's also serving as the coordination mechanism between instances to avoid thundering herd on cold start. That dual role is worth being explicit about in the design docs.

What's your deployment model? Are these pods sharing a volume, or is each pod's `/tmp` isolated? That changes how aggressive you can be with the local in-process layer.