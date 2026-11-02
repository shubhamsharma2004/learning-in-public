# Caching Fundamentals

**TL;DR:** A cache stores the results of expensive work (DB queries, API calls, computations) in fast storage so repeated requests are served quickly. The hard parts are deciding *what* to cache, *when* to invalidate it and *how* to keep it consistent.

## Key ideas

- **Cache hit / miss:** a hit is served from cache; a miss falls through to the source of truth and usually populates the cache.
- **TTL (time to live):** every entry expires after a set time, which bounds how stale data can get.
- **Eviction policies:** when memory is full, something must go. Common choices are **LRU** (least recently used), **LFU** (least frequently used) and **FIFO**.
- **Where caches live:** browser → CDN → application (in-memory) → distributed cache (Redis/Memcached) → database buffer pool.

## Caching strategies

| Strategy | How it works | Good for | Watch out for |
|---|---|---|---|
| **Cache-aside** (lazy loading) | App checks cache; on miss reads DB and writes to cache | Read-heavy workloads, most common pattern | First request is always slow; stale data until TTL |
| **Write-through** | Every write goes to cache and DB together | Data that is read right after it is written | Slower writes; caches data that may never be read |
| **Write-back** (write-behind) | Write to cache now, flush to DB later | Very write-heavy workloads | Risk of data loss if the cache node dies |
| **Read-through** | Cache itself loads from DB on a miss | Simplifies app code | Needs a cache layer that supports it |

## Trade-offs

- **Freshness vs. speed:** longer TTL means more hits but staler data.
- **Memory cost:** cache only hot data, not everything.
- **Cache stampede:** when a popular key expires, many requests hit the DB at once. Mitigate with locking, request coalescing or slightly randomised TTLs.
- **Invalidation:** on writes, delete the key rather than updating it, to avoid race conditions.

## Example: cache-aside with Redis (Python)

```python
import json
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_portfolio(user_id: int, db) -> dict:
    key = f"portfolio:{user_id}"
    cached = r.get(key)
    if cached:                      # cache hit
        return json.loads(cached)

    data = db.fetch_portfolio(user_id)  # cache miss -> source of truth
    r.setex(key, 300, json.dumps(data)) # store for 5 minutes
    return data

def update_portfolio(user_id: int, payload: dict, db) -> None:
    db.save_portfolio(user_id, payload)
    r.delete(f"portfolio:{user_id}")   # invalidate, next read refills
```

## Where I'd use it

In fintech dashboards, portfolio summaries and market data are read far more often than they change. Cache-aside with a short TTL (seconds to minutes) cuts database load heavily, while transactional data (orders, balances) should always be read from the source of truth.

## References

- [Redis documentation: caching patterns](https://redis.io/docs/latest/)
- [AWS: Caching best practices](https://aws.amazon.com/caching/best-practices/)
