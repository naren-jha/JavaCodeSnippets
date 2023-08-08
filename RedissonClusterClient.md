# Redisson
Redisson offers several advanced features beyond basic operations. Some of these features include:

1. Distributed Objects: Redisson provides distributed implementations of various Java objects such as Maps, Lists, Sets, Queues, and Locks. These objects can be used across different JVMs and allow for distributed data structures and synchronization.
With Redisson's distributed objects, you can create a distributed map that is shared across multiple JVMs. Each JVM can access and manipulate the map as if it were a local Java map, but the underlying data is stored in Redis and is available to all instances.

3. Data Caching: Redisson includes an advanced cache implementation that supports features like eviction policies, time-to-live (TTL) expiration, near-cache, and cache persistence. This allows for efficient caching of data in Redis with automatic data refreshing and eviction.

4. Distributed Locks and Semaphores: Redisson provides distributed locks and semaphores that can be used for distributed coordination and synchronization across multiple JVMs or instances. These locks and semaphores are built on Redis' atomic operations and are highly scalable and performant.

5. Pub/Sub Messaging: Redisson offers a robust Publish/Subscribe (Pub/Sub) messaging system that allows for real-time message-based communication between different components or services. It supports patterns, message listeners, and message filtering, making it suitable for building event-driven architectures.

6. Task Scheduling: Redisson includes a distributed task scheduler that enables scheduling and execution of tasks across multiple JVMs. This feature is useful for scenarios where tasks need to be executed in a distributed manner and coordinated across instances.

7. Remote Service Invocation: Redisson provides a remote service invocation framework that allows for invoking methods on remote objects across JVMs. It handles serialization, communication, and load balancing transparently, making it easier to build distributed systems.


# Setting up redisson client

## redisson maven dependency
First add redisson dpendency in pom.xml as below - 
```Java
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.19.2</version>
</dependency>
```

## redisson client configuration to work in cluster mode
Then write a redisson client as below -

```Java
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.client.codec.StringCodec;
import org.redisson.config.Config;
import org.redisson.config.ReadMode;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.util.logging.Level;
import java.util.logging.Logger;

import java.nio.charset.Charset;


@Component
public class RedissonClientHelper {

    @Value("${redis.cluster.nodes}")
    private String redisClusterNodes;

    @Value("${redis.cluster.connect.timeout}")
    private String redisClusterConnectTimeout;

    @Value("${redis.cluster.idle.connect.timeout}")
    private String redisClusterIdleConnectTimeout;

    @Value("${redis.cluster.timeout}")
    private String redisClusterTimeout;

    @Value("${redis.cluster.retry.attempts}")
    private String redisClusterRetryAttempt;

    @Value("${redis.cluster.slave.min.pool.idle.size}")
    private String redisClusterSlaveMinIdleSize;

    @Value("${redis.cluster.slave.max.pool.size}")
    private String redisClusterSlaveMaxPoolSize;

    @Value("${redis.cluster.master.min.pool.idle.size}")
    private String redisClusterMasterMinPoolSize;

    @Value("${redis.cluster.master.max.pool.size}")
    private String redisClusterMasterMaxPoolSize;

    @Value("${redisson.keep.alive}")
    private String redissonKeepAlive;

    @Value("${ping.connection.interval.milliseconds}")
    private String pingConnectionIntervalMilli;

    @Value("${reconnection.timeout.milliseconds}")
    private String reconnectionTimeoutMill;

    private static RedissonClient redissonClient;

    private static RedissonClientHelper instance;
    private static final Logger logger = Logger.getLogger(RedissonClientHelper.class.getName());

    @PostConstruct
    public void init(){
        initialize();

    }
    private void initialize(){
        logger.info("Default Charset: "+Charset.defaultCharset().name()+" with File Encoding: "+ System.getProperty("file.encoding"));
        logger.info("Redis Cluster Nodes: "+ redisClusterNodes);
        logger.info("Redis Cluster Connect Timeout: "+ reconnectionTimeoutMill);
        try {
            Config config = new Config();
            config.setCodec(new StringCodec())
                    .useClusterServers().addNodeAddress(redisClusterNodes)
                    .setConnectTimeout(Integer.parseInt(redisClusterConnectTimeout))
                    .setIdleConnectionTimeout(Integer.parseInt(redisClusterIdleConnectTimeout))
                    .setTimeout(Integer.parseInt(redisClusterTimeout))
                    .setReadMode(ReadMode.MASTER_SLAVE)
                    .setRetryAttempts(Integer.parseInt(redisClusterRetryAttempt))
                    .setSlaveConnectionMinimumIdleSize(Integer.parseInt(redisClusterSlaveMinIdleSize))
                    .setSlaveConnectionPoolSize(Integer.parseInt(redisClusterSlaveMaxPoolSize))
                    .setMasterConnectionMinimumIdleSize(Integer.parseInt(redisClusterMasterMinPoolSize))
                    .setMasterConnectionPoolSize(Integer.parseInt(redisClusterMasterMaxPoolSize))
                    .setKeepAlive(Configs.getBoolean(redissonKeepAlive))
                    .setPingConnectionInterval(Integer.parseInt(pingConnectionIntervalMilli))
                    .setRetryInterval(Integer.parseInt(reconnectionTimeoutMill));

            redissonClient = Redisson.create(config);
        } catch (Exception e){
            logger.log(Level.SEVERE,"Exception in Initializing Redisson Client {}", e);
        }
    }

    public RedissonClient getRedissonClient() {
        return redissonClient;
    }

}
```

## Working with redis Key-Value data structure using redisson
```Java
ObjectMapper objectMapper = new ObjectMapper();
String jsonValue = objectMapper.writeValueAsString(objectToStore);
redissonClientHelper.getRedissonClient().getBucket(key).set(jsonValue, ttl, TimeUnit.SECONDS);
```

The redissonClientHelper.getRedissonClient().getBucket(key) code is used for simple key-value operations.
In Redis, a "bucket" typically refers to a simple key-value data structure. The Redisson client provides the RBucket interface to work with simple key-value pairs. When you call getBucket(key) on the Redisson client, it returns an RBucket object associated with the specified key.

You can then perform various operations on the RBucket object, such as setting a value with a TTL using the set() method, retrieving the value using the get() method, or deleting the key-value pair using the delete() method.

```Java
import org.redisson.api.RBucket;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class RedissonKeyValueHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void setValue(String key, String value) {
        RBucket<String> bucket = redissonClient.getBucket(key);
        bucket.set(value);
    }

    public String getValue(String key) {
        RBucket<String> bucket = redissonClient.getBucket(key);
        return bucket.get();
    }

    public void deleteKey(String key) {
        RBucket<String> bucket = redissonClient.getBucket(key);
        bucket.delete();
    }
}
```

## Working with redis Hash data structure 
```Java
import org.redisson.api.RMap;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class RedissonMapHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void setHashValue(String key, String field, String value) {
        RMap<String, String> map = redissonClient.getMap(key);
        map.put(field, value);
    }

    public void putValueWithTTL(String key, String value, long ttl, TimeUnit timeUnit) {
        RMap<String, String> map = redissonClient.getMap("myMap");
        map.put(key, value);
        map.expire(key, ttl, timeUnit);
    }

    public String getHashValue(String key, String field) {
        RMap<String, String> map = redissonClient.getMap(key);
        return map.get(field);
    }

    public void deleteHashField(String key, String field) {
        RMap<String, String> map = redissonClient.getMap(key);
        map.remove(field);
    }
}
```

## Working with redisson RMapCache 
```Java
import org.redisson.api.RMapCache;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedissonMapCacheHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void putValue(String key, String value, long ttl, TimeUnit timeUnit) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");
        mapCache.put(key, value, ttl, timeUnit);
    }

    public String getValue(String key) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");
        return mapCache.get(key);
    }

    public void removeKey(String key) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");
        mapCache.remove(key);
    }
}
```

Using RMap you can set a TTL only for the entire map, but using RMapCache you can set a TTL either for the entire map or for the individual keys in the map. 

The main difference between RMap and RMapCache lies in the default behavior and additional features they provide:

**Default Behavior:**

* RMap: By default, entries in RMap do not have an expiration time. They will persist indefinitely until explicitly removed or updated.
* RMapCache: By default, entries in RMapCache can have an expiration time. When you store an entry in RMapCache, you can specify a TTL, and Redisson will automatically handle the expiration of entries based on the TTL.

**Additional Features:**

* RMap: RMap provides basic key-value storage operations without additional caching-related features such as eviction policies.
* RMapCache: RMapCache extends RMap and adds cache-related features. It supports features like time-based expiration, eviction policies (e.g., LRU, LFU), and cache-specific operations.

Redisson's RMapCache offers additional capabilities beyond basic key-value storage and TTL-based eviction. Some of the additional capabilities provided by RMapCache are:

1. Maximum Size Eviction: You can set a maximum size for the RMapCache, and when the size exceeds the configured threshold, Redisson automatically evicts the least recently used entries to make room for new entries. This feature is useful for cache management and ensuring that the cache doesn't grow indefinitely.

2. Entry Listener: Redisson allows you to attach entry listeners to RMapCache to receive notifications when specific events occur, such as the addition, removal, or expiration of entries. This enables you to react to changes in the cache and take necessary actions.

3. Asynchronous Operations: Redisson provides asynchronous versions of cache operations, allowing you to perform cache operations asynchronously, which can be beneficial for performance and scalability in certain scenarios.

4. Batch Operations: Redisson supports batch operations on RMapCache, allowing you to perform multiple cache operations in a single atomic batch. This can improve performance and reduce network round-trips when working with multiple cache entries simultaneously.

5. Cache Loader: Redisson allows you to configure a cache loader that is invoked automatically when a cache entry is not found. The cache loader can retrieve the missing entry from a data source, such as a database, and populate the cache, providing a seamless integration between the cache and external data sources.

### Max size eviction with RMapCache
```Java
import org.redisson.api.RMapCache;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedissonMapCacheHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void configureMapCacheWithEviction() {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");

        // Set maximum size and TTL for the map cache
        int maxSize = 100;
        long ttl = 10; // 10 seconds

        mapCache.setMaxSize(maxSize);
        mapCache.setMaxSizePolicy(RMapCache.MaxSizePolicy.PER_NODE);
        mapCache.setTimeToLive(ttl, TimeUnit.SECONDS);
    }

    public void putValue(String key, String value) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");
        mapCache.put(key, value);
    }

    // ...
}
```
The maxSize parameter in mapCache.setMaxSize(maxSize) represents the maximum size or capacity of the RMapCache. It specifies the maximum number of entries that can be stored in the cache before eviction is triggered.
When the number of entries in the RMapCache reaches the maxSize, Redisson applies the configured eviction policy to make room for new entries. The eviction policy determines which entries to remove when the cache is full.
The eviction policy set here is RMapCache.MaxSizePolicy.PER_NODE. This policy ensures that the maximum size is enforced per Redisson node rather than across the entire cluster.

Similarly, we have LRU eviction policy
```Java
import org.redisson.api.RMapCache;
import org.redisson.api.RedissonClient;
import org.redisson.client.codec.StringCodec;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedissonMapCacheHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void configureMapCacheWithLRUEviction(String mapCacheKey, int maxSize) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache(mapCacheKey);

        // Set maximum size and eviction policy to LRU
        mapCache.setMaxSize(maxSize);
        mapCache.setMaxSizePolicy(RMapCache.MaxSizePolicy.LRU);
    }

    public void putValue(String mapCacheKey, String key, String value) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache(mapCacheKey);
        mapCache.put(key, value);
    }

    // ...
}
```
The setMaxSize() method is used to set the maximum size of the cache, indicating the maximum number of entries that can be stored. The setMaxSizePolicy() method is then used to specify the eviction policy as LRU.

After configuring the LRU eviction policy, you can use the putValue() method to store key-value pairs in the RMapCache. Redisson will automatically handle the eviction of entries based on the LRU policy when the maximum size is reached. The least recently used entries will be evicted from the cache to make room for new entries.

#### All possible eviction policies:
Redisson's RMapCache supports several eviction policies that you can set using the setMaxSizePolicy() method. Here are the possible values for RMapCache.MaxSizePolicy:

1. **RMapCache.MaxSizePolicy.REJECT:** This is the default policy. It rejects new entries when the cache reaches its maximum size limit.
2. **RMapCache.MaxSizePolicy.PER_NODE:** Each Redisson node in a cluster or Redis deployment enforces its own maximum size independently. Entries will be evicted based on the maximum size specified per node.
3. **RMapCache.MaxSizePolicy.LFU:** Eviction is based on the Least Frequently Used (LFU) algorithm. Entries that are accessed less frequently will be evicted first when the cache reaches its maximum size.
4. **RMapCache.MaxSizePolicy.LRU:** Eviction is based on the Least Recently Used (LRU) algorithm. Entries that are accessed least recently will be evicted first when the cache reaches its maximum size.
5. **RMapCache.MaxSizePolicy.SIZE:** Eviction is based on the size of the entries in the cache. Entries with larger sizes will be evicted first when the cache reaches its maximum size.
6. **RMapCache.MaxSizePolicy.FREE_HEAP_RATIO:** Eviction is based on the ratio of free heap memory. Entries will be evicted when the free heap memory ratio falls below a certain threshold.


### Entry Expiry Listener with RMapCache
```Java
import org.redisson.api.RMapCache;
import org.redisson.api.RedissonClient;
import org.redisson.api.map.event.EntryExpiredListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedissonMapCacheHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void configureMapCacheWithExpirationListener() {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");

        // Set time-to-live (TTL) for entries
        long ttl = 60; // 60 seconds
        mapCache.setTimeToLive(ttl, TimeUnit.SECONDS);

        // Add entry expired listener
        mapCache.addListener(new EntryExpiredListener<String, String>() {
            @Override
            public void onExpired(String key, String value) {
                // Callback method when an entry expires
                handleExpiration(key);
            }
        });
    }

    private void handleExpiration(String key) {
        // Callback method to handle expiration
        System.out.println("Key expired: " + key);
        // Perform necessary actions or notify other components
        // ...
    }

    public void putValue(String key, String value) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");
        mapCache.put(key, value);
    }

    public void putValueWithTTL(String key, String value, long ttl, TimeUnit timeUnit) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");
        mapCache.put(key, value, ttl, timeUnit);
    }

    // ...
}
```

So when an entry expires in the RMapCache, it will execute callback method handleExpiration().

Using redisson, you can set TTL for entries in map. This feature is not provided by redis itself. In Redis, the key expiration is handled at the key level, not at the individual entry level within a map. When a key expires in Redis, the entire key and its associated values are removed.

However, Redisson provides additional functionality on top of Redis, including the ability to set TTLs for individual entries within a map using the RMapCache data structure. Redisson manages the expiration of individual entries in the map by using Redis' key expiration mechanism internally.

The entry listener feature in Redisson allows you to receive notifications when an entry in the RMapCache expires. It is a Redisson-specific feature that provides a convenient way to react to expiration events at the entry level.

Under the hood, Redisson leverages Redis' key expiration mechanism and combines it with internal bookkeeping to provide TTL-based eviction and event notification at the entry level within a map. This allows you to handle individual entry expiration events and perform custom actions based on those events.

You cannot achieve entry level expiry (and therefore notification) using simple RMap. for that you'll have to use RMapCache.

#### Expiring the entire map vs espiring entries in map
In both RMap and RMapCache, you can use the expire() method to set a TTL for the entire map or map cache. The TTL specifies how long the entire map or map cache will be retained before it is automatically expired and cleared.

The difference between RMap and RMapCache lies in their additional features and behaviors. RMapCache provides additional caching-related functionalities such as TTL-based entry expiration, eviction policies, and entry-level event notifications. On the other hand, RMap provides a simpler key-value storage structure without the caching-specific features.

#### Entry Add/Remove Listener with RMapCache
```Java
import org.redisson.api.RMapCache;
import org.redisson.api.RedissonClient;
import org.redisson.api.map.event.EntryAddedListener;
import org.redisson.api.map.event.EntryRemovedListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class RedissonMapCacheHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void addEntryAddedListener(String mapCacheKey) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache(mapCacheKey);

        // Register an entry added listener
        mapCache.addListener(new EntryAddedListener<String, String>() {
            @Override
            public void onAdded(String key, String value) {
                handleEntryAdded(key, value);
            }
        });
    }

    public void addEntryRemovedListener(String mapCacheKey) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache(mapCacheKey);

        // Register an entry removed listener
        mapCache.addListener(new EntryRemovedListener<String, String>() {
            @Override
            public void onRemoved(String key, String value) {
                handleEntryRemoved(key, value);
            }
        });
    }

    private void handleEntryAdded(String key, String value) {
        // Handle the entry added event
        System.out.println("Entry added - Key: " + key + ", Value: " + value);
        // Perform necessary actions or notify other components
        // ...
    }

    private void handleEntryRemoved(String key, String value) {
        // Handle the entry removed event
        System.out.println("Entry removed - Key: " + key + ", Value: " + value);
        // Perform necessary actions or notify other components
        // ...
    }

    // ...
}
```

### Asynchronous Operations using RMapCacheAsync
```Java
import org.redisson.api.RMapCacheAsync;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

@Component
public class RedissonMapCacheAsyncHelper {

    @Autowired
    private RedissonClient redissonClient;

    public CompletableFuture<Void> putValueAsync(String mapCacheKey, String key, String value, long ttl, TimeUnit timeUnit) {
        RMapCacheAsync<String, String> mapCacheAsync = redissonClient.getMapCache(mapCacheKey).async();

        return mapCacheAsync.putAsync(key, value, ttl, timeUnit)
                .thenAcceptAsync(v -> System.out.println("Value stored asynchronously with key: " + key));
    }

    public CompletableFuture<String> getValueAsync(String mapCacheKey, String key) {
        RMapCacheAsync<String, String> mapCacheAsync = redissonClient.getMapCache(mapCacheKey).async();

        return mapCacheAsync.getAsync(key)
                .thenApplyAsync(value -> {
                    System.out.println("Value retrieved asynchronously with key: " + key);
                    return value;
                });
    }

    // ...
}
```

The putValueAsync() method is used to store a value asynchronously in the RMapCache. It creates an RMapCacheAsync instance using the .async() method on the RMapCache object. Then, it uses the putAsync() method to asynchronously store the key-value pair with the specified TTL. The method returns a CompletableFuture<Void>, which allows you to handle the completion of the operation asynchronously. In this example, the completion of the operation is logged when the value is stored.

The getValueAsync() method is used to retrieve a value asynchronously from the RMapCache. It follows a similar pattern by creating an RMapCacheAsync instance and using the getAsync() method to asynchronously retrieve the value associated with the specified key. It returns a CompletableFuture<String> that represents the asynchronous result of the operation. In this example, the retrieved value is logged when it is available.

By using CompletableFuture and the asynchronous versions of cache operations provided by RMapCacheAsync, you can perform cache operations asynchronously, allowing for potentially improved performance and scalability in scenarios where parallelism and non-blocking operations are desired.


### Batch Operations using RMapCache and RBatch:
```Java
import org.redisson.api.RBatch;
import org.redisson.api.RMapCache;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class RedissonMapCacheBatchHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void performBatchOperations(String mapCacheKey) {
        RMapCache<String, String> mapCache = redissonClient.getMapCache(mapCacheKey);
        RBatch batch = redissonClient.createBatch();

        // Add multiple cache operations to the batch
        batch.getMapCache(mapCacheKey).putAsync("key1", "value1");
        batch.getMapCache(mapCacheKey).putAsync("key2", "value2");
        batch.getMapCache(mapCacheKey).putAsync("key3", "value3");
        // Add more cache operations as needed

        // Execute the batch operations
        batch.execute();

        // Access the results if needed
        // ...
    }

    // ...
}
```

The performBatchOperations() method sets up a batch context by creating an instance of RBatch using redissonClient.createBatch(). Then, multiple cache operations are added to the batch using putAsync() on the RMapCache instance retrieved from RBatch. These operations can include put, remove, or any other cache operation supported by RMapCache.

After adding all the cache operations to the batch, batch.execute() is called to execute all the operations atomically in a single network round-trip. This improves performance by reducing the number of network round-trips and provides atomicity for the batch operations.

If you need to access the results of individual cache operations within the batch, you can do so after calling batch.execute(). The batch execution will return the results, and you can process them accordingly.

By leveraging batch operations with RBatch, you can perform multiple cache operations on RMapCache in a single atomic batch, resulting in improved performance and reduced network overhead.

#### Using batch for get operation as well
```Java
import org.redisson.api.RBatch;
import org.redisson.api.RMapCache;
import org.redisson.api.RedissonClient;
import org.redisson.api.BatchResult;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.ExecutionException;

@Component
public class RedissonMapCacheBatchHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void performBatchGetOperations(String mapCacheKey) throws ExecutionException, InterruptedException {
        RMapCache<String, String> mapCache = redissonClient.getMapCache(mapCacheKey);
        RBatch batch = redissonClient.createBatch();

        // Add multiple cache get operations to the batch
        batch.getMapCache(mapCacheKey).getAsync("key1");
        batch.getMapCache(mapCacheKey).getAsync("key2");
        batch.getMapCache(mapCacheKey).getAsync("key3");
        // Add more cache get operations as needed

        // Execute the batch operations
        BatchResult<?> batchResult = batch.execute();

        // Access the results of individual cache get operations
        String value1 = (String) batchResult.getResponses().get(0).get();
        String value2 = (String) batchResult.getResponses().get(1).get();
        String value3 = (String) batchResult.getResponses().get(2).get();

        System.out.println("Value 1: " + value1);
        System.out.println("Value 2: " + value2);
        System.out.println("Value 3: " + value3);
    }

    // ...
}
```

The `batch.execute();` operation is an async operation. It initiates the execution of all the operations added to the batch, and it returns a BatchResult object immediately.

The BatchResult object provides a way to access the results of the individual operations performed in the batch. It contains a list of RFuture objects that represent the asynchronous results of each operation.

While the batch.execute() method itself is asynchronous and non-blocking, the blocking call to retrieve the results of the individual operations happens when you invoke the get() method on each RFuture object.

In the example code, when you call `batchResult.getResponses().get(0).get()`, it blocks the current thread until the result of the first operation becomes available. The get() method waits for the result and returns the value once it is ready.

Therefore, the blocking call occurs when you access the results of individual operations using the get() method. Keep in mind that if you're retrieving multiple results sequentially using get(), it may introduce blocking behavior that waits for each result to complete before proceeding to the next one.

If you want to handle the results asynchronously without blocking, you can use the thenAcceptAsync() or thenApplyAsync() methods on the RFuture objects to specify callbacks or transformation functions to process the results when they become available. This way, you can achieve non-blocking behavior and continue processing other tasks while waiting for the results to complete.

**What's the difference between thenAcceptAsync and thenApplyAsync ?**

The thenAcceptAsync() and thenApplyAsync() methods are both part of the RFuture interface in Redisson, which represents an asynchronous result of a Redisson operation. Both methods allow you to specify callbacks to handle the result of the operation asynchronously. However, there is a difference in their return types and intended usage:

1. thenAcceptAsync(): This method is used for asynchronous result handling without returning a value. It takes a Consumer functional interface as a parameter, which represents a callback function that accepts the result as input and performs some action. The thenAcceptAsync() method returns a CompletableFuture<Void> that represents the completion of the callback action.

```Java
RFuture<String> future = ...;
future.thenAcceptAsync(result -> {
    // Handle the result asynchronously
    // Perform some action with the result
});
```

2. thenApplyAsync(): This method is used for asynchronous result handling with the ability to transform the result. It takes a Function functional interface as a parameter, which represents a transformation function that accepts the result as input, performs some computation, and returns a new value. The thenApplyAsync() method returns a CompletableFuture<U> where U is the type of the transformed result.
```Java
RFuture<String> future = ...;
CompletableFuture<Integer> transformedFuture = future.thenApplyAsync(result -> {
    // Transform the result asynchronously
    // Perform some computation based on the result and return a new value
    return result.length();
});
```

In summary, thenAcceptAsync() is used when you want to perform some action asynchronously with the result, without returning a value. On the other hand, thenApplyAsync() is used when you want to perform a computation asynchronously based on the result and return a transformed value.

Both methods allow you to handle the results asynchronously without blocking the current thread, providing flexibility in handling the asynchronous outcomes of Redisson operations.



## Working with redis List data structure 
```Java
import org.redisson.api.RList;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class RedissonListHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void addToList(String key, String value) {
        RList<String> list = redissonClient.getList(key);
        list.add(value);
    }

    public String getFromList(String key, int index) {
        RList<String> list = redissonClient.getList(key);
        return list.get(index);
    }

    public void removeFromList(String key, String value) {
        RList<String> list = redissonClient.getList(key);
        list.remove(value);
    }

    public int getListSize(String key) {
        RList<String> list = redissonClient.getList(key);
        return list.size();
    }
}
```

## Working with redis Set data structure
```Java
import org.redisson.api.RSet;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class RedissonSetHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void addToSet(String key, String value) {
        RSet<String> set = redissonClient.getSet(key);
        set.add(value);
    }

    public boolean isMemberOfSet(String key, String value) {
        RSet<String> set = redissonClient.getSet(key);
        return set.contains(value);
    }

    public void removeFromSet(String key, String value) {
        RSet<String> set = redissonClient.getSet(key);
        set.remove(value);
    }

    public int getSetSize(String key) {
        RSet<String> set = redissonClient.getSet(key);
        return set.size();
    }
}
```

## Working with redis SortedSet data structure
```Java
import org.redisson.api.RSortedSet;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class RedissonSortedSetHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void addToSortedSet(String key, String value, double score) {
        RSortedSet<String> sortedSet = redissonClient.getSortedSet(key);
        sortedSet.add(score, value);
    }

    public boolean isMemberOfSortedSet(String key, String value) {
        RSortedSet<String> sortedSet = redissonClient.getSortedSet(key);
        return sortedSet.contains(value);
    }

    public void removeFromSortedSet(String key, String value) {
        RSortedSet<String> sortedSet = redissonClient.getSortedSet(key);
        sortedSet.remove(value);
    }

    public int getSortedSetSize(String key) {
        RSortedSet<String> sortedSet = redissonClient.getSortedSet(key);
        return sortedSet.size();
    }
}
```

## Working with redisson distributed lock
```Java
import org.redisson.Redisson;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedissonLockHelper {

    @Autowired
    private RedissonClient redissonClient;

    public void performTaskWithLock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // Acquire the lock
            lock.lock();

            // Perform your task here
            // ...

        } finally {
            // Release the lock
            lock.unlock();
        }
    }

    public boolean tryLock(String lockKey, long timeoutSeconds) {
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // Try to acquire the lock with a timeout
            return lock.tryLock(timeoutSeconds, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }

    public void releaseLock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        if (lock.isLocked()) {
            lock.unlock();
        }
    }
}
```

The performTaskWithLock() method shows an example of performing a task while holding the lock. It acquires the lock using lock.lock() before executing the task and releases the lock using lock.unlock() in a finally block to ensure the lock is always released, even in case of exceptions.

The tryLock() method demonstrates trying to acquire the lock with a timeout. It attempts to acquire the lock using lock.tryLock(timeout, TimeUnit) and returns true if the lock is acquired within the specified timeout. If the lock acquisition is interrupted, it returns false.

The releaseLock() method is used to explicitly release the lock. It checks if the lock is still held (isLocked()) and calls unlock() to release it.


## Using Redisson's distributed pub/sub feature
```Java
// Create a Redisson client
RedissonClient redisson = Redisson.create();

// Get a distributed topic instance
RTopic<String> topic = redisson.getTopic("myDistributedTopic");

// Subscribe to the topic
topic.addListener(String.class, (channel, message) -> {
    System.out.println("Received message: " + message);
});

// Publish a message to the topic
topic.publish("Hello, Redisson!");

// Close the Redisson client
redisson.shutdown();
```

## Redisson vs Lettuce
<img width="736" alt="image" src="https://github.com/naren-jha/JavaCodeSnippets/assets/58611230/df432fd4-5d09-4a3e-a92e-bb5067d03c41">

## Using distributed task (across JVM) in redisson

Here's an example that demonstrates how to use Redisson's distributed task scheduling feature:
```Java
// Create a Redisson client
RedissonClient redisson = Redisson.create();

// Get a distributed task scheduler
RScheduledExecutorService executorService = redisson.getExecutorService("myExecutorService");

// Define a task to be executed
Runnable task = () -> {
    System.out.println("Executing task...");
    // Perform the task logic here
};

// Schedule the task to be executed every 5 seconds
// initial delay is - 0 secs
// period is - 5 secs
RScheduledFuture<?> scheduledFuture = executorService.scheduleAtFixedRate(task, 0, 5, TimeUnit.SECONDS);

// Wait for some time to observe task execution
Thread.sleep(20000);

// Cancel the task
scheduledFuture.cancel();

// Shut down the Redisson client
redisson.shutdown();
```
In this example, Redisson's RScheduledExecutorService represents a distributed task scheduler. You can obtain an instance of the scheduler using redisson.getExecutorService("myExecutorService"). The scheduleAtFixedRate method is used to schedule a task (represented by the Runnable interface) to be executed every 5 seconds.

The task logic, defined within the Runnable, will be executed by Redisson across multiple JVM instances, ensuring distributed task execution and coordination.

After scheduling the task, we wait for some time (20 seconds in this example) to observe the task execution. Finally, the scheduled task is canceled using scheduledFuture.cancel(), and the Redisson client is shut down.

Redisson's distributed task scheduling feature allows you to schedule and execute tasks in a distributed manner across multiple JVM instances. This is particularly useful in scenarios where tasks need to be coordinated across instances or when you want to leverage the computing power of multiple machines to execute tasks concurrently.

It's important to note that Redisson's distributed task scheduling supports various scheduling options, including fixed-rate scheduling, fixed-delay scheduling, cron-based scheduling, and more. You can choose the appropriate scheduling strategy based on your specific requirements.

#### this can be used as replacement for spring scheduler
Redisson's distributed task scheduling feature can serve as a suitable replacement for the Spring Scheduler in scenarios where you require coordination and uniqueness of scheduled tasks across multiple JVM instances.

While the Spring Scheduler provides task scheduling within a single JVM instance, Redisson's distributed task scheduler extends this capability to a distributed environment, allowing you to schedule and execute tasks across multiple JVM instances. It ensures that tasks are coordinated and executed uniquely across the distributed system, providing consistency and reliability.

By leveraging Redisson's distributed task scheduling feature, you can overcome the limitations of scheduling tasks within a single JVM and achieve distributed task execution with ease. This is especially valuable in scenarios where you have a microservices architecture or a distributed system where tasks need to be executed across multiple instances.

Redisson's distributed task scheduler not only handles coordination and uniqueness but also offers advanced scheduling options, task cancellation, error handling, and various execution strategies. It provides a comprehensive solution for managing and executing tasks in a distributed environment.

So, if you require task scheduling with coordination and uniqueness across JVM instances, Redisson's distributed task scheduler can be a powerful and reliable alternative to the Spring Scheduler. It enhances the capabilities of task scheduling in a distributed system and enables you to build robust and scalable applications.

#### Fixed-Rate Scheduling:
```Java
RScheduledFuture<?> scheduledFuture = executorService.scheduleAtFixedRate(task, initialDelay, period, timeUnit);
```
This schedules the task to be executed at a fixed rate, where initialDelay specifies the delay before the first execution, period represents the time duration between consecutive executions, and timeUnit denotes the unit of time for the delay and period values.

#### Fixed-Delay Scheduling:
```Java
RScheduledFuture<?> scheduledFuture = executorService.scheduleWithFixedDelay(task, initialDelay, delay, timeUnit);
```
This schedules the task to be executed with a fixed delay between the completion of one execution and the start of the next. initialDelay indicates the delay before the first execution, delay represents the delay between consecutive executions, and timeUnit specifies the unit of time for the delay values.

#### Cron-Based Scheduling:
```Java
RScheduledFuture<?> scheduledFuture = executorService.scheduleCron(task, cronExpression);
```
This schedules the task to be executed based on a cron expression. The cronExpression parameter defines the specific schedule pattern for the task execution, following the cron syntax.

##### Difference between Fixed-Rate Scheduling and Fixed-Delay Scheduling:
The main difference between fixed-rate scheduling and fixed-delay scheduling is how the time intervals between task executions are determined:

1. Fixed-Rate Scheduling: In fixed-rate scheduling, the period between task executions is constant, regardless of how long the task takes to execute. The next execution will start at the specified period after the previous execution completes, regardless of any delays that may have occurred. This means that tasks will be executed at a fixed rate, maintaining a consistent interval between consecutive executions.

2. Fixed-Delay Scheduling: In fixed-delay scheduling, the delay between the completion of one task execution and the start of the next execution is constant. The next execution will start after the specified delay has passed since the completion of the previous execution. This means that the delay remains constant regardless of the time it takes for the task to execute.

To illustrate the difference, let's consider an example:

Suppose you have a task that takes 2 seconds to execute, and you schedule it with:
```Java
executorService.scheduleAtFixedRate(task, 0, 5, TimeUnit.SECONDS);
```
With fixed-rate scheduling, the task will be executed every 5 seconds, regardless of its execution time. So, even if the task takes 2 seconds to complete, the next execution will start after 5 seconds from the previous start time.

On the other hand, if you schedule the same task with fixed-delay scheduling:
```Java
executorService.scheduleWithFixedDelay(task, 0, 5, TimeUnit.SECONDS);
```
In this case, the delay between the completion of one execution and the start of the next execution is fixed at 5 seconds. So, if the task takes 2 seconds to execute, the next execution will start after 5 seconds from the completion of the previous execution. This ensures a consistent delay between consecutive task executions.

In summary, the difference lies in how the time intervals are determined:

- Fixed-Rate Scheduling: Consistent period between task executions, regardless of execution time.
- Fixed-Delay Scheduling: Consistent delay between the completion of one execution and the start of the next execution, regardless of execution time.

