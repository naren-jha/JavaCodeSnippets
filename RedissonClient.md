First add redisson dpendency in pom.xml as below - 
```Java
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.19.2</version>
</dependency>
```

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

# Working with key-value data structure using redisson
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

