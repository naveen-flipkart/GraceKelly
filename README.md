#GraceKelly
####a best effort cache synchronization library for distributed systems

GraceKelly is a best effort cache synchronization library designed to
shield distributed systems and services from direct exposure to
unpredictable request loads. It improves load and response SLA
predictability in distributed environments. It also enables graceful
degradation with stale data as fallback, in a degraded distributed
ecosystem.

###Why is it needed?

####A chaotic place Any big distributed environment is inherently
complex and chaotic. This complexity arises due to the complex
dependencies between different services. The variability of the
requests and responses that this environment is exposed to makes it a
chaotic place

<img src="https://img3a.flixcart.com//www/promos/new/20130905-115236-soa.png">

This chaos means the predictability of load and latency is
reduced. This makes the environment and it’s SLAs vulnerable to
arbitrary request loads. It’s necessary to shield the environment from
such externally triggered unpredictability. Since service SLAs are
affected by service load, such shielding also ensures their
predictability. This means one must systemically strive to hold on to
as much predictability as possible when building a service/system.

####Sheilds up

Caches act as sentinels in an distributed environment. Although their
primary function is to reduce latency, when used appropriately they
excel at bringing predictability to a system. That’s because a cache
request is extremely predictable, with almost no variability, either
in response times or the load per request. One could say that there is
positive co-relation between the percentage of Cache hits and the
predictability of a system/environment.

<img src="https://img1a.flixcart.com//www/promos/new/20130905-115342-soa-cached.png">

####Cache expiry

Every time there is a cache miss the environment and SLAs become a
little bit more vulnerable. In this context, the common cache usage
pattern of expiry based on a ttl and subsequent re-population seems
risky. Using cache expiry as a proxy/trigger for cache synchronization
exposes the underlying system to potentially harmful request pattern
load for the duration of synchronization.

- **t0** – a heavily requested cache entry c1 expires
- **t1** – there is a cache miss for c1 and a request is sent to the service to fulfill
- **t2** – the cache has been repopulated with c2

The time between **t1** and **t2** is the duration of exposure. The
predictability of the target service and all the services it depends
on during this time is affected by the the per request load and the
qps of all requests that result in a cache miss for c1.

What would be good to have is a cache library with regular caching
semantics but one that accommodates refreshing a cache entry rather
than expiring it based on ttl. This is exactly what GraceKelly is,
it’s inspired by Gooogle Guava’s LoadingCache.

###What does it do?

GraceKelly tries it’s best to refresh the cache entry that has
expired. The refresh lifecycle is purely request triggered and doesn’t
monitor/maintain the cache. For every request

- It looks up the cache and returns the value if a cache entry is present.
- If the returned cache entry has expired it dispatches a task to refresh the cache entry.
- If for some reason the refresh fails, it can extend the ttl of the existing entry or do nothing.

Note that a cache entry is never removed(though it can be evicted by
size constraints).

This does two things.

- Shields the backend services and systems from exposure to unnecessary request load.  
- Decouples response SLAs from backend degradation and availability concerns, there by allowing for graceful degradation with stale data as fallback.

###The Library

The library has a single Class **Kelly** that takes implementations of
two different interfaces a **CacheProvider** and a
**CacheLoader**. They passaround a **CacheEntry**

####CacheProvider

The CacheProvider interface is used to implement adapters to different cache implementations where the cached values
are finally persisted and retrieved from. For eg: one would implement a CacheProvider for couchbase or memcached.


```java

public interface CacheProvider <T>{
    /**
     * Returns a {@link CacheEntry}<T> if it is present in the underlying cache,
     * or it returns a null otherwise.
     * @param key
     * @return {@link CacheEntry}<T> for the given key or return null if not present
     * @throws CacheProviderException
     */
    CacheEntry<T> get(String key) throws CacheProviderException;

    /**
     * Tries to update the cache with the given {@link CacheEntry}<T> for the given key
     * @param key
     * @param value
     * @return true or false based on the success of putting the {@link CacheEntry}<T>
     * into the cache.
     * @throws CacheProviderException
     */
    Boolean put(String key, CacheEntry<T> value) throws CacheProviderException;
}
```

####CacheLoader

The CacheLoader provides a single method to reload cache, based on an existing entry in the cache.
The implementation of CacheLoader should be able to reload the cache given the key of the and the
previous value of the CacheEntry.

```java
public interface CacheLoader<T> {

    /**
     * Takes a {@link String} key and a value/Object of type <T> and returns a
     * {@link CacheEntry}<T>. The implementation of this method is supposed to
     * return the CacheEntry with the latest Value for the given key.
     * @param key
     * @param prevValue
     * @return {@link CacheEntry} of the type parameter specified during
     * declaration of this instance of CacheLoader
     * @throws CacheLoaderException
     */
    public CacheEntry<T> reload(String key, T prevValue) throws CacheLoaderException;
}
```

####CacheEntry
The CacheEntry class is a simple java object that holds data required to get, put and invalidate a
cache entry. The generic parameter indicates the type of the object that'll be store against the
given key. usage is as follows, where the **ttl is in seconds**

```java
//cache entry valid for 5 minutes since time of creation
CacheEntry<SomeObject> cacheEntry = new CacheEntry<SomeObject>("key", someObject, 300);

String key = cacheEntry.getKey() //returns the key of the CacheEntry
SomeObject = cahceEntry.getValue() //returns value of the CacheEntry
long ttl = cacheEntry.getTtl() //returns the ttl in seconds
```
