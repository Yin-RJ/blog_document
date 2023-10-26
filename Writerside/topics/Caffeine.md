# Caffeine

## 添加策略
### 手动加载
```Java
    @Test
    public void testManualPopulation() {
        Cache<Integer, Integer> cache = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .maximumSize(10)
            .build();

        // 从缓存中获取数据，如果没有数据返回null
        Integer num = cache.getIfPresent(1);
        Assertions.assertNull(num);

        // 从缓存中获取数据，如果没有就生成缓存元素，如果无法生成则返回null
        num = cache.get(1, e -> 5);
        Assertions.assertEquals(5, num);

        // 添加或更新一个缓存元素
        cache.put(1, 2);
        cache.put(2, 5);
        Assertions.assertEquals(2, cache.getIfPresent(1));
        Assertions.assertEquals(5, cache.getIfPresent(2));

        // 驱逐一个缓存元素
        cache.invalidate(1);
        Assertions.assertNull(cache.getIfPresent(1));
    }
```
Cache接口提供了显式搜索查找、更新和移除缓存元素的能力。

在手动加载缓存元素的时候可以尽量使用`cache.get(key, key -> value)
`操作，当缓存元素不存在的时候，会进行计算并直接写入缓存内，如果缓存元素存在，则直接返回缓存元素。当缓存元素生成失败的时候，该方法也会返回null。

### 自动加载
```Java
    /**
     * 自动加载
     */
    @Test
    public void testLoadingPopulation() {
        LoadingCache<Integer, Integer> loadingCache = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .maximumSize(10)
            .build(this::generateValue);

        // 获取缓存元素，如果不存在则生成，生成失败则返回null
        Integer num = loadingCache.get(1);
        Assertions.assertEquals(1, num);

        try {
            num = loadingCache.get(2);
            Assertions.assertNull(num);
        } catch (Exception e) {
            // nothing
        }

        Map<Integer, Integer> map = new HashMap<>();
        List<Integer> keys = Arrays.asList(4, 5, 6, 9, 2);
        try {
            // 批量查找缓存，不存在则插入，一旦发生异常，所有的插入都会失败
            map = loadingCache.getAll(keys);
        } catch (Exception e) {
            Assertions.assertEquals(0, map.size());
        }

        keys = Arrays.asList(4, 5, 6, 9);

        map = loadingCache.getAll(keys);
        Assertions.assertEquals(4, map.size());
        Assertions.assertEquals(4, map.get(4));
    }

    private Integer generateValue(int key) {
        if (key == 2) {
            throw new RuntimeException();
        }
        return key;
    }
```
在Cache的基础上增加CacheLoader能力后就形成了LoadingCache。可以通过`getAll(Iterable<? extends K> 
keys)`方法实现提前缓存一批key。

### 手动异步加载
```Java
    /**
     * 手动异步加载
     * @throws ExecutionException
     * @throws InterruptedException
     */
    @Test
    public void testAsyncCache() throws ExecutionException, InterruptedException {
        AsyncCache<Integer, Integer> cache = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .maximumSize(10)
            .buildAsync();

        // 获取缓存元素，如果没有则返回null
        CompletableFuture<Integer> numFuture = cache.getIfPresent(1);
        Assertions.assertNull(numFuture);

        CompletableFuture<Integer> contentFuture = cache.get(1, this::generateValue);
        Assertions.assertNotNull(contentFuture);
        Assertions.assertEquals(1, contentFuture.get());

        CompletableFuture<Integer> future = CompletableFuture.completedFuture(5);
        cache.put(2, future);
        Assertions.assertEquals(5, cache.getIfPresent(2).get());

        // 可以获取到map来直接操作
        ConcurrentMap<Integer, CompletableFuture<Integer>> map = cache.asMap();
        map.put(6, future);
        Assertions.assertEquals(5, cache.getIfPresent(6).get());

        // synchronous()方法会直接返回一个Cache对象
        cache.synchronous().invalidate(1);
    }
```
AsyncCache是Cache的一个变体，提供了在Executor上生成缓存并返回CompletableFuture
的能力，可以在构造AsyncCache对象的时候指定线程池。

### 自动异步加载
```Java
AsyncLoadingCache<Key, Graph> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    // 你可以选择: 去异步的封装一段同步操作来生成缓存元素
    .buildAsync(key -> createExpensiveGraph(key));
    // 你也可以选择: 构建一个异步缓存元素操作并返回一个future
    .buildAsync((key, executor) -> createExpensiveGraphAsync(key, executor));

// 查找缓存元素，如果其不存在，将会异步进行生成
CompletableFuture<Graph> graph = cache.get(key);
// 批量查找缓存元素，如果其不存在，将会异步进行生成
CompletableFuture<Map<Key, Graph>> graphs = cache.getAll(keys);
```
一个 AsyncLoadingCache是一个 AsyncCache 加上 AsyncCacheLoader能力的实现。

在需要同步的方式去生成缓存元素的时候，CacheLoader是合适的选择。而在异步生成缓存的场景下， AsyncCacheLoader则是更合适的选择并且它会返回一个 CompletableFuture。

通过 getAll可以达到批量查找缓存的目的。 默认情况下，在getAll 方法中，将会对每个不存在对应缓存的key调用一次 AsyncCacheLoader.asyncLoad 来生成缓存元素。 在批量检索比单个查找更有效率的场景下，你可以覆盖并开发AsyncCacheLoader.asyncLoadAll 方法来使你的缓存更有效率。

值得注意的是，你可以通过实现一个 AsyncCacheLoader.asyncLoadAll并在其中为没有在参数中请求的key也生成对应的缓存元素。打个比方，如果对应某个key生成的缓存元素与包含这个key的一组集合剩余的key所对应的元素一致，那么在asyncLoadAll中也可以同时加载剩下的key对应的元素到缓存当中。

## 驱逐策略
### 基于容量
### 基于时间
### 基于引用类型