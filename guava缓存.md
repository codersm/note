---
title: guava缓存
date: 2017-08-01 15:02:42
tags: guava cache
---
# Guava Cache

缓存在各种用例中非常有用。例如，当一个值计算或检索的代价很高时，您应该考虑使用高速缓存，并且您需要在某个输入上多次使用它的值。

Cache类似于ConcurrentMap，但不完全相同。最根本的区别是，ConcurrentMap会持续添加到其中的所有元素，直到它们被显式删除；另一方面，缓存通常被配置为自动驱逐数据，以便限制其内存占用。在某些情况下，由于其自​​动缓存加载即使不驱逐数据，LoadingCache可能会有用的。

一般来说，Guava缓存实用程序适用于：

* 牺牲空间获取时间提升；
* 多次获取keys；
* 缓存不需要存储比RAM内容更多的数据。

## CacheLoader

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) throws AnyException {
               return createExpensiveGraph(key);
             }
           });

...
try {
  return graphs.get(key);
} catch (ExecutionException e) {
  throw new OtherException(e.getCause());
}
```

The canonical way to query a LoadingCache is with the method get(K). This will either return an already cached value, or else use the cache's CacheLoader to atomically load a new value into the cache. Because CacheLoader might throw an Exception, LoadingCache.get(K) throws ExecutionException. (If the cache loader throws an unchecked exception, get(K) will throw an UncheckedExecutionException wrapping it.) You can also choose to use getUnchecked(K), which wraps all exceptions in UncheckedExecutionException, but this may lead to surprising behavior if the underlying CacheLoader would normally throw checked exceptions.

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .expireAfterAccess(10, TimeUnit.MINUTES)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return createExpensiveGraph(key);
             }
           });

...
return graphs.getUnchecked(key);
```

* getAll(Iterable<? extends K>)

  批量查找。默认情况下，getAll将为缓存中缺少的每个键单独调用CacheLoader.load，当批量检索比许多单独的查找更有效时，您可以覆盖CacheLoader.loadAll。

## Callable

```java
Cache<Key, Value> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .build(); // look Ma, no CacheLoader
...
try {
  // If the key wasn't in the "easy to compute" group, we need to
  // do things the hard way.
  cache.get(key, new Callable<Value>() {
    @Override
    public Value call() throws AnyException {
      return doThingsTheHardWay(key);
    }
  });
} catch (ExecutionException e) {
  throw new OtherException(e.getCause());
}
```

## Inserted Directly

* cache.put(key, value)

* Cache.asMap()

## Eviction

Guava provides three basic types of eviction: size-based eviction, time-based eviction, and reference-based eviction.

* Size-based Eviction

  * CacheBuilder.maximumSize(long)
  * CacheBuilder.weigher(Weigher)
  * CacheBuilder.maximumWeight(long)

* Timed Eviction

  * expireAfterAccess(long, TimeUnit)
  * expireAfterWrite(long, TimeUnit)

* Reference-based Eviction

Guava allows you to set up your cache to allow the garbage collection of entries, by using weak references for keys or values, and by using soft references for values.

 * CacheBuilder.weakKeys()
 * CacheBuilder.weakValues()
 * CacheBuilder.softValues()

## Explicit Removals

* individually, using Cache.invalidate(key)
* in bulk, using Cache.invalidateAll(keys)
* to all entries, using Cache.invalidateAll()

## Removal Listeners

您可以通过CacheBuilder.removalListener（RemovalListener）来为缓存指定删除侦听器，以便在删除条目时执行某些操作。

> 注意，RemovalListener抛出的任何异常都会被记录（使用Logger）并被吞噬。

```java
CacheLoader<Key, DatabaseConnection> loader = new CacheLoader<Key, DatabaseConnection> () {
  public DatabaseConnection load(Key key) throws Exception {
    return openConnection(key);
  }
};
RemovalListener<Key, DatabaseConnection> removalListener = new RemovalListener<Key, DatabaseConnection>() {
  public void onRemoval(RemovalNotification<Key, DatabaseConnection> removal) {
    DatabaseConnection conn = removal.getValue();
    conn.close(); // tear down properly
  }
};

return CacheBuilder.newBuilder()
  .expireAfterWrite(2, TimeUnit.MINUTES)
  .removalListener(removalListener)
  .build(loader);
```

默认情况下同步执行删除监听器操作，并且由于通常在正常缓存操作期间执行缓存维护，因此昂贵的删除监听器可能会降低正常缓存功能。如果您有一个昂贵的删除监听器，请使用RemovalListeners.isynchronous（RemovalListener，Executor）来装饰RemovalListener以异步操作。

## When Does Cleanup Happen

使用CacheBuilder构建的缓存不会自动执行清除和排除值，或者在值过期后立即执行任何值。相反，它在写入操作期间执行少量的维护，或者在偶尔的读取操作期间，如果写入是罕见的。

The reason for this is as follows: if we wanted to perform Cache maintenance continuously, we would need to create a thread, and its operations would be competing with user operations for shared locks. Additionally, some environments restrict the creation of threads, which would make CacheBuilder unusable in that environment.

Instead, we put the choice in your hands. If your cache is high-throughput, then you don't have to worry about performing cache maintenance to clean up expired entries and the like. If your cache does writes only rarely and you don't want cleanup to block cache reads, you may wish to create your own maintenance thread that calls Cache.cleanUp() at regular intervals.

If you want to schedule regular cache maintenance for a cache which only rarely has writes, just schedule the maintenance using ScheduledExecutorService.

## Refresh 【*】

* LoadCache.refresh（K）

刷新与驱逐不一样。如在LoadCache.refresh（K）中指定的，刷新键可能会异步加载键的新值。在刷新的过程中仍旧返回旧值（如果有），与驱逐相反，强制检索等待，直到重新加载该值。

如果在刷新时抛出异常，则会保留旧值，并记录并吞入异常。

CacheLoader可以通过覆盖CacheLoader.reload（K，V）来指定刷新时使用的智能行为，从而允许您在计算新值时使用旧值。

```java
// Some keys don't need refreshing, and we want refreshes to be done asynchronously.
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
.maximumSize(1000)
.refreshAfterWrite(1, TimeUnit.MINUTES)
.build(
new CacheLoader<Key, Graph>() {
    public Graph load(Key key) { // no checked exception
    return getGraphFromDatabase(key);
    }

    public ListenableFuture<Graph> reload(final Key key, Graph prevGraph) {
    if (neverNeedsRefresh(key)) {
        return Futures.immediateFuture(prevGraph);
    } else {
        // asynchronous!
        ListenableFutureTask<Graph> task = ListenableFutureTask.create(new Callable<Graph>() {
        public Graph call() {
            return getGraphFromDatabase(key);
        }
        });
        executor.execute(task);
        return task;
    }
    }
});
```

* refreshAfterWrite（自动刷新）

与expireAfterWrite相反，refreshAfterWrite将使一个关键字在指定的持续时间后可以刷新，但刷新将仅在查询条目时才实际启动。

## 其他特性

* Statistics（统计）

By using CacheBuilder.recordStats(), you can turn on statistics collection for Guava caches. The Cache.stats() method returns a CacheStats object.

## Interruption

缓存加载方法（如Cache.get）不会抛出InterruptedException。我们也可以让这些方法支持InterruptedException，但这种支持注定是不完备的，并且会增加所有使用者的成本，而只有少数使用者实际获益。详情请继续阅读。

Cache.get请求到未缓存的值时会遇到两种情况：当前线程加载值；或等待另一个正在加载值的线程。这两种情况下的中断是不一样的。等待另一个正在加载值的线程属于较简单的情况：使用可中断的等待就实现了中断支持；但当前线程加载值的情况就比较复杂了：因为加载值的CacheLoader是由用户提供的，如果它是可中断的，那我们也可以实现支持中断，否则我们也无能为力。

如果用户提供的CacheLoader是可中断的，为什么不让Cache.get也支持中断？从某种意义上说，其实是支持的：如果CacheLoader抛出InterruptedException，Cache.get将立刻返回（就和其他异常情况一样）；此外，在加载缓存值的线程中，Cache.get捕捉到InterruptedException后将恢复中断，而其他线程中InterruptedException则被包装成了ExecutionException。

原则上，我们可以拆除包装，把ExecutionException变为InterruptedException，但这会让所有的LoadingCache使用者都要处理中断异常，即使他们提供的CacheLoader不是可中断的。如果你考虑到所有非加载线程的等待仍可以被中断，这种做法也许是值得的。但许多缓存只在单线程中使用，它们的用户仍然必须捕捉不可能抛出的InterruptedException异常。即使是那些跨线程共享缓存的用户，也只是有时候能中断他们的get调用，取决于那个线程先发出请求。

对于这个决定，我们的指导原则是让缓存始终表现得好像是在当前线程加载值。这个原则让使用缓存或每次都计算值可以简单地相互切换。如果老代码（加载值的代码）是不可中断的，那么新代码（使用缓存加载值的代码）多半也应该是不可中断的。

如上所述，Guava Cache在某种意义上支持中断。另一个意义上说，Guava Cache不支持中断，这使得LoadingCache成了一个有漏洞的抽象：当加载过程被中断了，就当作其他异常一样处理，这在大多数情况下是可以的；但如果多个线程在等待加载同一个缓存项，即使加载线程被中断了，它也不应该让其他线程都失败（捕获到包装在ExecutionException里的InterruptedException），正确的行为是让剩余的某个线程重试加载。为此，我们记录了一个bug。然而，与其冒着风险修复这个bug，我们可能会花更多的精力去实现另一个建议AsyncLoadingCache，这个实现会返回一个有正确中断行为的Future对象。


**源码分析**

创建缓存对象，为啥会触发ClassLoader?

Applies a supplemental hash function to a given hash code, which defends against poor quality hash functions. This is critical when the concurrent hash map uses power-of-two length hash tables, that otherwise encounter collisions for hash codes that do not differ in lower or upper bits.

```java
static int rehash(int h) {
  // Spread bits to regularize both segment and index locations,
  // using variant of single-word Wang/Jenkins hash.
  // TODO(kevinb): use Hashing/move this to Hashing?
  h += (h << 15) ^ 0xffffcd7d;
  h ^= (h >>> 10);
  h += (h << 3);
  h ^= (h >>> 6);
  h += (h << 2) + (h << 14);
  return h ^ (h >>> 16);
}

V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
  checkNotNull(key);
  checkNotNull(loader);
  try {
    if (count != 0) { // read-volatile
      // don't call getLiveEntry, which would ignore loading values
      ReferenceEntry<K, V> e = getEntry(key, hash);
      if (e != null) {
        long now = map.ticker.read();
        V value = getLiveValue(e, now);
        if (value != null) {
          recordRead(e, now);
          statsCounter.recordHits(1);
          return scheduleRefresh(e, key, hash, value, now, loader);
        }
        ValueReference<K, V> valueReference = e.getValueReference();
        if (valueReference.isLoading()) {
          return waitForLoadingValue(e, key, valueReference);
        }
      }
    }

    // at this point e is either null or expired;
    return lockedGetOrLoad(key, hash, loader);
  } catch (ExecutionException ee) {
    Throwable cause = ee.getCause();
    if (cause instanceof Error) {
      throw new ExecutionError((Error) cause);
    } else if (cause instanceof RuntimeException) {
      throw new UncheckedExecutionException(cause);
    }
    throw ee;
  } finally {
    postReadCleanup();
  }
}

```

