# Caching in .NET

*A deep-dive walkthrough of caching in .NET — covering `IMemoryCache` for in-process caching, `IDistributedCache` and Redis for caching shared across multiple instances, the cache-aside pattern implemented concretely in C#, expiration policies (absolute vs. sliding) and eviction callbacks, cache stampede prevention via `GetOrCreateAsync`'s locking behavior, ASP.NET Core's Output Caching middleware, the new `HybridCache` API unifying in-memory and distributed caching, and how these concrete .NET APIs map onto the architectural concepts this series' Distributed Cache system design guide covers more abstractly.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [IMemoryCache: In-Process Caching](#1-imemorycache-in-process-caching)
3. [Absolute vs. Sliding Expiration](#2-absolute-vs-sliding-expiration)
4. [Eviction Callbacks and Cache Entry Size](#3-eviction-callbacks-and-cache-entry-size)
5. [The Cache-Aside Pattern, Implemented Concretely](#4-the-cache-aside-pattern-implemented-concretely)
6. [Cache Stampede Prevention: GetOrCreateAsync's Locking](#5-cache-stampede-prevention-getorcreateasyncs-locking)
7. [IDistributedCache: Caching Shared Across Instances](#6-idistributedcache-caching-shared-across-instances)
8. [Redis via IDistributedCache and StackExchange.Redis Directly](#7-redis-via-idistributedcache-and-stackexchangeredis-directly)
9. [Serialization: What Actually Goes Into a Distributed Cache](#8-serialization-what-actually-goes-into-a-distributed-cache)
10. [HybridCache: Unifying In-Memory and Distributed Caching](#9-hybridcache-unifying-in-memory-and-distributed-caching)
11. [Output Caching: Caching Whole HTTP Responses](#10-output-caching-caching-whole-http-responses)
12. [Cache Invalidation in Practice](#11-cache-invalidation-in-practice)
13. [Choosing Between IMemoryCache, IDistributedCache, and HybridCache](#12-choosing-between-imemorycache-idistributedcache-and-hybridcache)
14. [Common Pitfalls](#13-common-pitfalls)
15. [Quick Reference Table](#quick-reference-table)
16. [Conclusion](#conclusion)

---

## Introduction

This series' Distributed Cache system design guide covers caching's architectural concerns in depth — partitioning, replication, eviction policy trade-offs, invalidation as the domain's central hard problem, and cache stampede mitigation as general concepts applicable to any language or platform. This guide goes the other direction: the concrete .NET APIs that actually implement those concepts in a real ASP.NET Core application — `IMemoryCache` for a single process's own memory, `IDistributedCache` (commonly backed by Redis) for a cache shared safely across every instance of a horizontally-scaled application, and the newer `HybridCache` that combines both into a single, coherent API. Every architectural concept that system design guide introduces (TTL, eviction, stampede protection, cache-aside) reappears here as a specific method call or configuration option, and this guide cross-references that one directly rather than re-deriving the underlying reasoning.

```plaintext
IMemoryCache      → LOCAL to one process — fastest, but each instance of a
                       horizontally-scaled app has its OWN, separate cache
IDistributedCache → SHARED across every instance — a network hop away
                       (typically Redis), but consistent across the whole deployment
HybridCache        → BOTH layers together — an in-process L1 cache backed by a
                        shared L2 distributed cache, per this series' Distributed Cache
                        guide's Section 13 multi-layer caching discussion
```

---

## 1. IMemoryCache: In-Process Caching

### The basic get/set API

```csharp
public class ProductService
{
    private readonly IMemoryCache _cache;
    public ProductService(IMemoryCache cache) => _cache = cache; // registered via AddMemoryCache(), SINGLETON

    public async Task<Product> GetProductAsync(int id)
    {
        if (_cache.TryGetValue($"product:{id}", out Product? cached))
            return cached!; // CACHE HIT — no database call at all

        var product = await _repository.GetByIdAsync(id); // CACHE MISS
        _cache.Set($"product:{id}", product, TimeSpan.FromMinutes(10));
        return product;
    }
}
```

`IMemoryCache` stores data as ordinary .NET objects, directly in the current process's own memory — no serialization, no network hop, genuinely the fastest possible cache access, but scoped entirely to *this one running instance*. Registered via `builder.Services.AddMemoryCache()`, it's a Singleton by nature (per this series' ASP.NET Core Dependency Injection guide's Section 4 lifetime reasoning — a cache genuinely needs to be shared across every request within one process, which is exactly what Singleton provides).

### `GetOrCreate`/`GetOrCreateAsync`: the idiomatic, single-call cache-aside shorthand

```csharp
public async Task<Product> GetProductAsync(int id)
{
    return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
    {
        entry.SlidingExpiration = TimeSpan.FromMinutes(10); // Section 2 covers this in depth
        return await _repository.GetByIdAsync(id); // only runs on a genuine MISS
    })!;
}
```

This collapses the `TryGetValue`/miss/`Set` sequence from Section 1's first example into one call — the factory delegate only executes on a genuine cache miss, and Section 5 covers a genuinely important detail about *how* concurrent misses for the same key are handled by this specific method.

---

## 2. Absolute vs. Sliding Expiration

### Absolute expiration: a fixed point in time, regardless of access pattern

```csharp
_cache.Set(key, value, new MemoryCacheEntryOptions
{
    AbsoluteExpiration = DateTimeOffset.UtcNow.AddMinutes(30) // expires at a FIXED moment, no matter what
});
// or, relative to now:
_cache.Set(key, value, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
});
```

An absolute expiration guarantees an entry is never cached for longer than the specified duration, *regardless* of how frequently it's accessed — this directly implements this series' Distributed Cache guide's Section 5 principle that TTL bounds maximum staleness independent of any other factor, which matters specifically for data where "this might be up to 30 minutes stale" is an acceptable, bounded guarantee you want to hold firm.

### Sliding expiration: resets on every access, expiring only after a period of genuine inactivity

```csharp
_cache.Set(key, value, new MemoryCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(10) // expires 10 minutes after the LAST access, not the first
});
```

A sliding expiration keeps an entry alive as long as it's being actively used, expiring only once it genuinely stops being accessed for the specified window — this is the right choice for data where "keep it cached as long as it's genuinely useful" matters more than a hard, fixed staleness bound, but worth knowing it can, in principle, keep an entry cached indefinitely if it's accessed frequently enough, never actually reflecting a genuinely long-since-changed underlying value.

### Combining both, and why that combination is often the genuinely correct default

```csharp
_cache.Set(key, value, new MemoryCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(5),           // stays alive while actively used...
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1) // ...but NEVER longer than this, regardless
});
```

This combination closes sliding expiration's one real gap — a popular entry keeps resetting its sliding window and staying alive, but the absolute expiration still guarantees it's periodically refreshed from the source of truth no less often than the absolute bound, which is frequently the genuinely correct default for real-world caching needs: keep hot data hot, but never let anything go *too* stale regardless of how popular it remains.

---

## 3. Eviction Callbacks and Cache Entry Size

### Reacting to an entry being removed, for whatever reason

```csharp
var options = new MemoryCacheEntryOptions()
    .RegisterPostEvictionCallback((key, value, reason, state) =>
    {
        Console.WriteLine($"Entry {key} evicted: {reason}"); // Expired, Removed, Replaced, Capacity, or TokenExpired
    });
_cache.Set(key, value, options);
```

`PostEvictionCallback` fires whenever an entry leaves the cache, for any reason — worth knowing this exists for cases needing to react to eviction specifically (releasing an associated unmanaged resource, per this series' Memory Management guide's `IDisposable` discipline, or logging for diagnostic purposes), though it's a genuinely specialized tool most everyday caching code doesn't need.

### `SizeLimit` and per-entry `Size`: bounding the cache's overall memory footprint

```csharp
builder.Services.AddMemoryCache(options => options.SizeLimit = 1024); // the cache's TOTAL size budget

_cache.Set(key, value, new MemoryCacheEntryOptions { Size = 1 }); // this ENTRY counts as "1" against that budget
```

This directly implements this series' Distributed Cache guide's Section 5 eviction discussion — `IMemoryCache` doesn't automatically know how "big" any given object actually is in bytes, so `SizeLimit` and each entry's own declared `Size` establish an abstract unit system you define (could be a literal byte count, or a simpler unit like "1 per entry, cap at 10,000 entries") that the cache uses to decide when it's full and needs to start evicting.

### Priority: influencing WHICH entries get evicted first once the cache is under pressure

```csharp
_cache.Set(key, value, new MemoryCacheEntryOptions { Priority = CacheItemPriority.High }); // evicted LAST
```

`CacheItemPriority` (`Low`, `Normal`, `High`, `NeverRemove`) is a hint influencing eviction order once `SizeLimit` pressure forces the cache to reclaim space — worth knowing `NeverRemove` exists and is a genuinely dangerous option to reach for casually, since it opts a specific entry out of memory-pressure-driven eviction entirely, which can undermine the whole point of having a size limit if overused.

---

## 4. The Cache-Aside Pattern, Implemented Concretely

### This series' Distributed Cache guide's Section 7 pattern, as actual, runnable C#

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product:{id}";

    if (_cache.TryGetValue(cacheKey, out Product? cached))
        return cached!; // HIT — served entirely from cache, no database touched

    var product = await _repository.GetByIdAsync(id); // MISS — read from the origin
    _cache.Set(cacheKey, product, TimeSpan.FromMinutes(10)); // populate the cache for NEXT time
    return product;
}

public async Task UpdateProductAsync(Product product)
{
    await _repository.UpdateAsync(product); // write to the ORIGIN first
    _cache.Remove($"product:{product.Id}");   // then INVALIDATE — per this series' Distributed
                                                 //  Cache guide's Section 6 write-invalidate discussion
}
```

This is exactly the pattern this series' Distributed Cache guide's Section 7 describes architecturally, made concrete — the read path checks the cache first, falling back to the origin and populating the cache on a miss; the write path always writes through to the origin directly, then invalidates (rather than updates) the corresponding cache entry, letting the next read repopulate it correctly rather than risking a race where the cache is updated with a value that's already stale by the time it's written.

---

## 5. Cache Stampede Prevention: GetOrCreateAsync's Locking

### The problem, exactly as this series' Distributed Cache guide's Section 8 describes it

```plaintext
If TEN concurrent requests all miss the SAME cache key at once (right
  after it expired, or on first-ever access), a NAIVE cache-aside
  implementation would trigger TEN separate, redundant calls to the
  origin — the exact "thundering herd" this series' Distributed Cache
  guide's Section 8 warns can overwhelm an origin store sized only for
  cached traffic.
```

### `GetOrCreateAsync`'s built-in per-key locking closes this automatically

```csharp
public async Task<Product> GetProductAsync(int id)
{
    return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
    {
        entry.SlidingExpiration = TimeSpan.FromMinutes(10);
        return await _repository.GetByIdAsync(id); // per this series' async/await guide's Section 1 —
                                                       //  this genuinely I/O-bound call is what gets COALESCED
    })!;
}
```

This is worth knowing as a genuinely important, easy-to-miss implementation detail: `IMemoryCache.GetOrCreateAsync` internally uses per-key locking, so if ten concurrent calls request the same, currently-uncached key simultaneously, only the *first* actually executes the factory delegate (the database call) — the other nine wait on the same in-flight operation and receive its result once it completes, rather than each independently hitting the origin. This is precisely the "single-flight" / request-coalescing pattern this series' Distributed Cache guide's Section 8 recommends, and it's built into the standard method, not something you need to implement yourself for this specific API.

---

## 6. IDistributedCache: Caching Shared Across Instances

### The problem `IMemoryCache` structurally cannot solve: consistency across multiple running instances

```plaintext
Per this series' High-Volume Transaction Processing guide's own scaling
  discussion: a horizontally-scaled application has MULTIPLE instances
  running simultaneously, each with its OWN, entirely separate
  IMemoryCache — instance A caching a value tells instance B NOTHING;
  each instance independently misses and independently hits the origin,
  and worse, different instances can serve DIFFERENT, inconsistent
  cached values for the SAME key at the SAME time.
```

### The interface, and the byte-array-based API shape

```csharp
public interface IDistributedCache
{
    byte[]? Get(string key);
    Task<byte[]?> GetAsync(string key, CancellationToken token = default);
    void Set(string key, byte[] value, DistributedCacheEntryOptions options);
    Task SetAsync(string key, byte[] value, DistributedCacheEntryOptions options, CancellationToken token = default);
    void Remove(string key);
    Task RemoveAsync(string key, CancellationToken token = default);
    void Refresh(string key); // resets a SLIDING expiration without re-fetching the value
    Task RefreshAsync(string key, CancellationToken token = default);
}
```

Worth noting the crucial, structural difference from `IMemoryCache` immediately: `IDistributedCache` works exclusively in `byte[]` — because a distributed cache backend (Redis, per Section 7) is a genuinely separate process, potentially on a different machine entirely, there's no way to store a live .NET object reference in it; everything must be serialized to bytes first, which is Section 8's whole subject and a real, structural cost `IMemoryCache` simply doesn't have.

### Extension methods providing a string-based convenience layer

```csharp
string? cachedJson = await _distributedCache.GetStringAsync(cacheKey);
await _distributedCache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product), options);
```

`GetStringAsync`/`SetStringAsync` (from `Microsoft.Extensions.Caching.Distributed`) are convenience extension methods handling the `byte[]`-to-`string` conversion for you — you still need to serialize your actual object to and from that string yourself (typically as JSON, per Section 8), but this saves the manual UTF-8 encoding/decoding step.

---

## 7. Redis via IDistributedCache and StackExchange.Redis Directly

### Registering Redis as the IDistributedCache implementation

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp:"; // prefixes every key — useful for sharing one Redis instance across apps
});
```

This registers Redis as the concrete backend behind `IDistributedCache` — application code injecting `IDistributedCache` doesn't need to know or care that Redis specifically is behind it (exactly the same abstraction-over-provider benefit this series' Logging guide's Section 1 and Configuration guide's Section 1 both establish for their own respective systems) — swapping to a different `IDistributedCache` implementation later requires touching only this registration, not any of the code consuming the interface.

### When to drop down to `StackExchange.Redis` directly, bypassing `IDistributedCache`

```csharp
var redis = ConnectionMultiplexer.Connect(connectionString);
IDatabase db = redis.GetDatabase();
await db.StringIncrementAsync("counter:visits"); // an ATOMIC INCREMENT — IDistributedCache has NO equivalent
await db.SetAddAsync("tags:product:42", "featured"); // Redis SET operations — also no IDistributedCache equivalent
```

`IDistributedCache`'s interface is deliberately minimal — a simple key-value get/set/remove contract, portable across genuinely different backend technologies (Redis, SQL Server, NCache). For genuinely Redis-specific capabilities this series' Rate Limiter guide's Section 4 and Distributed Cache guide both rely on directly (atomic `INCR`, sorted sets, pub/sub), `IDistributedCache`'s abstraction doesn't expose them at all — using `StackExchange.Redis`'s own `IDatabase` API directly is the correct choice once you genuinely need Redis-specific functionality, accepting the loss of backend-portability in exchange for it.

---

## 8. Serialization: What Actually Goes Into a Distributed Cache

### Every value must be serialized to bytes before storage, and deserialized back on read

```csharp
public async Task<Product?> GetProductAsync(int id)
{
    var cachedBytes = await _distributedCache.GetAsync($"product:{id}");
    if (cachedBytes is null) return null; // MISS

    return JsonSerializer.Deserialize<Product>(cachedBytes); // DESERIALIZE back into a real object
}

public async Task SetProductAsync(Product product)
{
    var bytes = JsonSerializer.SerializeToUtf8Bytes(product); // SERIALIZE before storing
    await _distributedCache.SetAsync($"product:{product.Id}", bytes,
        new DistributedCacheEntryOptions { SlidingExpiration = TimeSpan.FromMinutes(10) });
}
```

This is precisely the real, structural cost this series' Distributed Cache guide's Section 10 discusses in the abstract — serialization/deserialization is genuine CPU overhead paid on every distributed cache operation, unlike `IMemoryCache`'s direct object reference storage — and choosing a fast serialization format (JSON is the common, convenient default; a binary format like MessagePack is meaningfully faster for genuinely hot paths) is a real, deliberate performance decision, exactly as that guide's Section 10 frames the serialization-format choice generally.

### Why the cached TYPE needs to remain compatible across deployments

```plaintext
If the Product class's SHAPE changes between deployments (a property
  renamed or removed) while EXISTING cached entries (serialized under
  the OLD shape) are still present in Redis, deserializing them against
  the NEW class definition can fail or silently produce incorrect
  data — worth planning for, especially across a rolling/blue-green
  deployment where OLD and NEW application versions may briefly run SIMULTANEOUSLY,
  both reading and writing the SAME distributed cache.
```

Worth knowing this as a genuinely real operational concern for distributed caching specifically (an `IMemoryCache` entry simply doesn't survive a deployment at all, sidestepping this problem entirely) — versioning cache keys (`product:v2:42`) or designing DTOs deliberately tolerant of missing/added fields (this series' API Versioning guide's Section 3 Tolerant Reader pattern, applied here to cache payloads rather than API responses) are both legitimate mitigations.

---

## 9. HybridCache: Unifying In-Memory and Distributed Caching

### The problem HybridCache solves: implementing the two-layer pattern from scratch is real, repeated work

```plaintext
Per this series' Distributed Cache guide's Section 13: a genuinely
  well-tuned caching setup often wants BOTH a fast, local in-process
  layer (IMemoryCache) AND a shared, consistent distributed layer
  (IDistributedCache) — checking the local cache first, falling back to
  the distributed cache, falling back to the origin, and keeping BOTH
  layers populated correctly, is real, non-trivial code to hand-write
  correctly, especially once Section 5's stampede protection needs to
  span BOTH layers together.
```

### HybridCache: one API, handling both layers and stampede protection together

```csharp
builder.Services.AddHybridCache(); // wires up BOTH an in-memory L1 and a DISTRIBUTED L2 (if IDistributedCache is registered)

public class ProductService
{
    private readonly HybridCache _cache;
    public ProductService(HybridCache cache) => _cache = cache;

    public async Task<Product> GetProductAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"product:{id}", async cancellationToken =>
            await _repository.GetByIdAsync(id), // the factory — same shape as Section 5's IMemoryCache pattern
            cancellationToken: default);
    }
}
```

Introduced as a stable API in .NET 9, `HybridCache` genuinely unifies Sections 1 and 6-7 into a single, coherent API — it automatically maintains both a fast local (`IMemoryCache`-backed) layer and, if an `IDistributedCache` is also registered, a shared distributed layer behind it, checking local first, then distributed, then falling back to your factory delegate — while also extending Section 5's per-key stampede protection to span *both* layers and, critically, across *multiple application instances* (something `IMemoryCache`'s own per-process locking structurally cannot do, since it has no visibility into what other instances are doing).

### Tag-based invalidation: a capability neither `IMemoryCache` nor `IDistributedCache` provides on its own

```csharp
await _cache.GetOrCreateAsync($"product:{id}", factory, tags: ["products", $"category:{categoryId}"]);

await _cache.RemoveByTagAsync("category:5"); // invalidates EVERY cached entry tagged with THIS category,
                                                //  regardless of individual key — genuinely new capability
```

This is a genuinely useful capability worth highlighting on its own — invalidating every cache entry associated with a given category (or any other grouping) in one call, without needing to know or track every individual key that might be affected, is something neither `IMemoryCache` nor plain `IDistributedCache` offers natively, and directly addresses a real, common real-world invalidation need this series' Distributed Cache guide's Section 6 discussion doesn't have a ready-made API answer for at the architectural level alone.

---

## 10. Output Caching: Caching Whole HTTP Responses

### A fundamentally different layer from everything else in this guide: caching the RESPONSE, not application data

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("ProductsPolicy", policy => policy.Expire(TimeSpan.FromMinutes(5)).Tag("products"));
});

app.UseOutputCache();

app.MapGet("/products/{id}", GetProduct).CacheOutput("ProductsPolicy");
```

Everything covered so far in this guide caches application *data* — a `Product` object, say — that your own code then uses to construct a response. Output Caching instead caches the fully-rendered HTTP *response itself*, at the middleware layer (per this series' Middleware guide's own pipeline model) — a subsequent identical request can be served the cached response directly, without the endpoint's own code (and, by extension, any application-level caching or database work within it) ever running at all.

### Cache invalidation for output caching, via the same tag mechanism

```csharp
await _outputCacheStore.EvictByTagAsync("products", default); // invalidates every CACHED RESPONSE tagged "products"
```

This mirrors Section 9's `HybridCache` tag-based invalidation directly, just applied at the response-caching layer instead of the data-caching layer — worth knowing both systems share this same tag-based invalidation *concept*, even though they're genuinely separate caching mechanisms operating at different layers of the application.

---

## 11. Cache Invalidation in Practice

### Every invalidation strategy this series' Distributed Cache guide's Section 6 covers has a direct, concrete .NET implementation

```plaintext
TTL-based (Section 2 of THIS guide): AbsoluteExpiration/SlidingExpiration
Explicit, write-path invalidation (Section 4's cache-aside pattern):
  _cache.Remove(key) / await _distributedCache.RemoveAsync(key)
Tag-based invalidation (Section 9): HybridCache's RemoveByTagAsync
Event-driven invalidation (per this series' Distributed Cache guide's
  Section 6 and Notification System guide's fan-out pattern): a message
  consumer, subscribed to a domain event, calling one of the above
  removal methods in reaction — no different mechanically from any
  other event-driven .NET code this series covers elsewhere
```

This section exists specifically to close the loop between that guide's architectural vocabulary and this guide's concrete APIs — every strategy that guide describes as a general concept maps onto one of these specific, callable .NET methods, and choosing which strategy to use for a given piece of cached data is exactly the same deliberate, per-data-type decision that guide's Section 6 frames it as.

---

## 12. Choosing Between IMemoryCache, IDistributedCache, and HybridCache

### The decision, stated directly

```plaintext
Single-instance application, or data that's genuinely fine being
  slightly inconsistent across instances → IMemoryCache alone.
Horizontally-scaled application needing CONSISTENT cached data across
  every instance → IDistributedCache (Redis), or HybridCache for the
  performance benefit of a local L1 layer on top of it.
New development, .NET 9+, wanting BOTH layers with unified stampede
  protection and tag-based invalidation without hand-rolling the
  two-layer logic yourself → HybridCache, registered with BOTH
  AddMemoryCache-equivalent AND AddStackExchangeRedisCache underneath it.
```

This is the practical resolution of this series' Distributed Cache guide's Section 13 multi-layer caching discussion, in concrete .NET terms — `HybridCache` is, for new development, generally the recommended starting point specifically because it gives you the local-cache performance benefit and the distributed-cache consistency guarantee together, with stampede protection and tag invalidation already handled, rather than needing to choose one now and potentially re-architect toward the other layer later.

---

## 13. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Using `IMemoryCache` alone in a horizontally-scaled application, expecting consistency | Each instance has its own, entirely separate cache — different instances can serve different, stale values for the same key simultaneously | Use `IDistributedCache`/Redis (or `HybridCache`) whenever consistency across multiple instances genuinely matters (Section 6) |
| Manually implementing cache-aside without using `GetOrCreateAsync`'s built-in locking | Reintroduces the exact cache stampede problem the built-in method already solves for you | Use `GetOrCreateAsync` rather than hand-rolling `TryGetValue`/miss/`Set` for anything where stampede protection matters (Section 5) |
| Forgetting that `IDistributedCache` requires serialization | Code written against `IMemoryCache`'s direct object storage doesn't port over directly — a genuine, real cost is easy to overlook when switching | Budget for serialization overhead explicitly; choose a fast format for genuinely hot paths (Section 8) |
| Changing a cached type's shape across a deployment without considering already-cached entries | Deserialization failures or silently incorrect data when old cached entries meet a new class definition, especially during rolling deployments | Version cache keys or design cached DTOs to tolerate shape changes gracefully (Section 8) |
| Using `CacheItemPriority.NeverRemove` casually | Opts an entry out of memory-pressure-driven eviction entirely, which can undermine the cache's own size limit if overused | Reserve `NeverRemove` for genuinely justified, rare cases; let ordinary priority levels govern eviction order otherwise (Section 3) |
| Reaching for `IDistributedCache`'s minimal interface when genuinely Redis-specific operations are needed | `IDistributedCache` has no equivalent for atomic increments, sets, or pub/sub — forcing awkward workarounds | Use `StackExchange.Redis`'s `IDatabase` directly once genuinely Redis-specific capability is needed, accepting the portability trade-off (Section 7) |
| Confusing output caching with application-data caching | They operate at genuinely different layers (the whole HTTP response vs. a piece of application data) with different invalidation needs | Understand output caching (Section 10) as complementary to, not a replacement for, data-level caching covered in the rest of this guide |
| Hand-rolling a two-layer local+distributed cache from scratch on .NET 9+ | Real, repeated implementation work re-solving a problem `HybridCache` already handles, including cross-instance stampede protection | Default to `HybridCache` for new development needing both layers (Section 9, Section 12) |

---

## Quick Reference Table

| API | Storage | Scope | Serialization Needed |
|---|---|---|---|
| `IMemoryCache` | In-process objects | One instance only | No |
| `IDistributedCache` | Bytes, in a backend (Redis, etc.) | Shared across all instances | Yes |
| `HybridCache` | Both (local + distributed) | Local fast path + shared consistency | Yes, for the distributed layer |
| Output Caching | Full HTTP responses | Configurable (per this series' Middleware guide's pipeline) | N/A (raw response bytes) |

| Concept | .NET Mechanism |
|---|---|
| Absolute expiration | `AbsoluteExpiration`/`AbsoluteExpirationRelativeToNow` |
| Sliding expiration | `SlidingExpiration` |
| Stampede prevention | `GetOrCreateAsync`'s built-in per-key locking |
| Explicit invalidation | `.Remove(key)` / `.RemoveAsync(key)` |
| Tag-based invalidation | `HybridCache.RemoveByTagAsync(tag)` |
| Redis-specific operations | `StackExchange.Redis`'s `IDatabase`, bypassing `IDistributedCache` |

---

## Conclusion

The concrete .NET caching APIs map directly onto the architectural concepts this series' Distributed Cache system design guide covers — `IMemoryCache`'s absolute/sliding expiration is TTL made concrete, `GetOrCreateAsync`'s per-key locking is that guide's request-coalescing stampede mitigation already built in, and the cache-aside pattern this guide implements in actual C# is exactly the read/write flow that guide describes at the architectural level. The one genuinely new axis this guide adds is the in-process-versus-distributed distinction itself: `IMemoryCache`'s speed comes specifically from never leaving the process and never touching a byte of serialization, while `IDistributedCache`'s consistency across a horizontally-scaled deployment comes specifically from accepting both of those costs — and `HybridCache` exists precisely because most real, well-tuned applications want both properties simultaneously, at different layers, rather than being forced to choose one globally.

Knowing which of the three to reach for — and knowing that stampede protection, TTL, and invalidation are the same underlying concerns regardless of which one you choose — is what turns "add caching" from a vague performance aspiration into a specific, well-reasoned implementation choice, grounded in the same trade-offs this series' Distributed Cache guide establishes architecturally and made concrete here as actual, callable .NET APIs.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the two-instances-serving-different-cached-values-from-IMemoryCache-alone incident that made the case for IDistributedCache better than any architecture diagram ever could.*
