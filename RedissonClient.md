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

You can set a TTL for a specific key in both RMap and RMapCache using the expire() method. This allows you to have time-based expiration for individual keys in both data structures.

The main difference between RMap and RMapCache lies in the default behavior and additional features they provide:

**Default Behavior:**

* RMap: By default, entries in RMap do not have an expiration time. They will persist indefinitely until explicitly removed or updated.
* RMapCache: By default, entries in RMapCache can have an expiration time. When you store an entry in RMapCache, you can specify a TTL, and Redisson will automatically handle the expiration of entries based on the TTL.

**Additional Features:**

* RMap: RMap provides basic key-value storage operations without additional caching-related features such as eviction policies.
* RMapCache: RMapCache extends RMap and adds cache-related features. It supports features like time-based expiration, eviction policies (e.g., LRU, LFU), and cache-specific operations.


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
