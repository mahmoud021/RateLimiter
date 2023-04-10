## RateLimiter
### What is rate limiting in software systems?
Rate limiting is an API’s configured behavior to reject the sender’s request when the number of requests over a window of time crosses a limit or threshold. The sender here could be another system identified using a user id, client id, IP address, or other such identifier.

### Why do software systems need rate limiting ?
If a public API allowed its users to make an unlimited number of requests per hour, it could lead to :
- resource exhaustion
- decreasing quality of the service
- denial of service attacks

This might result in a situation where the service is unavailable or slow. It could also lead to more unexpected costs being incurred by the service.any system should always protect itself against tremendous rps as it directly impacts its availability, cost,latency and it’s overall experience. 

it is clear that rate-limiter as a design choice is fit for following use cases:
- Keep infrastructure cost as per the planned budget.
- Gracefully reject client’s request in case of high load.
- Implement different revenue model for different kind of user’s subscription.
- Security against attack and suspicious activities

## Design for our usecase

As shown in the  diagram, the rate limiter is part of the application and will cap the requests on the server based on only the individual server’s state where it is running.
![1623139562756](https://user-images.githubusercontent.com/122011790/230999895-cbe30dc0-248d-44d1-8a76-630208ca00fb.png)

## The Token Bucket Algorithm
Token Bucket is an algorithm that you can use to implement rate limiting. In short, it works as follows:
- A bucket is created with a certain capacity (number of tokens).
- When a request comes in, the bucket is checked. If there is enough capacity, the request is allowed to proceed. Otherwise, the request is denied.
- When a request is allowed, the capacity is reduced.
- After a certain amount of time, the capacity is replenished.

### Token Bucket Algorithm Concepts
![-viber-2022-01-07-18-44-42-191](https://user-images.githubusercontent.com/122011790/231000651-c35c969b-ca62-4bed-afdb-f7e7cfa7204d.png)

Bucket: As you can see, he has a fixed volume of tokens (if you set 1000 tokens into our bucket, this is the maximum value of volume).  
Refiller: Regularly fills missing tokens based on bandwidth management to Bucket (calling every time before Consume).  
Consume: Takes away tokens from our Bucket (take away 1 token or many tokens -- usually it depends on the weight of calling consume method, it is a customizable and flexible variable, but in 99% of cases, we need to consume only one token).  

Below, you can see an example of a refiller working with Bandwidth management to refresh tokens every minute:

![refill](https://user-images.githubusercontent.com/122011790/231002444-9d2729dd-1ad2-498e-b277-c332c7778315.jpg)

Consume (as action) takes away Tokens from Bucket.

### Buket Size
The bucket is needed for storing a current count of Tokens, maximum possible count of tokens, and refresh time to generate a new token.
The Token Bucket algorithm has fixed memory for storing Bucket, and it consists of the following variables:
The volume of Bucket (maximum possible count of tokens) - 8 bytes
The current count of tokens in a bucket - 8 bytes
The count of nanoseconds for generating a new token - 8 bytes
The header of the object: 16 bytes

In total: 40 bytes
For example, in one gigabyte, we can store 25 million buckets. It’s very important to know because, usually, we store information about our buckets in caches and consequently into RAM (Random Access Memory).

## Bucket4J
Bucket4j is the most popular library in Java-World for realizing rate-limiting features ,Bucket4J is a Java rate-limiting library based on a token-bucket algorithm.
### example

```java
public class Example {

   public static void main(String args[]) {

       //Create the Bandwidth to set the rule - one token per minute
       Bandwidth oneCosumePerMinuteLimit = Bandwidth.simple(1, Duration.ofMinutes(1));

       //Create the Bucket and set the Bandwidth which we created above
       Bucket bucket = Bucket.builder()
                               .addLimit(oneCosumePerMinuteLimit)
                               .build();

       //Call method tryConsume to set count of Tokens to take from the Bucket,
       //returns boolean, if true - consume successful and the Bucket had enough Tokens inside Bucket to execute method tryConsume
       System.out.println(bucket.tryConsume(1)); //return true

       //Call method tryConsumeAndReturnRemaining and set count of Tokens to take from the Bucket
       //Returns ConsumptionProbe, which include much more information than tryConsume, such as the
       //isConsumed - is method consume successful performed or not, if true - is successful
       //getRemainingTokens - count of remaining Tokens
       //getNanosToWaitForRefill - Time in nanoseconds to refill Tokens in our Bucket
       ConsumptionProbe consumptionProbe = bucket.tryConsumeAndReturnRemaining(1);
       System.out.println(consumptionProbe.isConsumed()); //return false since we have already called method tryConsume, but Bandwidth has  a limit with rule - one token per one minute
       System.out.println(consumptionProbe.getRemainingTokens()); //return 0, since we have already consumed all of the Tokens
       System.out.println(consumptionProbe.getNanosToWaitForRefill()); //Return around 60000000000 nanoseconds
   }
   
```
## Manage Collection of bukets 
now we want  to create buket for each user based on ad identifer:
so we can do that :
```java
 class UsersRateLimiter {
   ConcurrentHashMap<String, Buket> usersBuketsCache = new ConcurrentHashMap<String, Buket>();
   //code
}
```
so each user will have his own buket , and we use ConcurrentHashMap is a thread-safe implementation of the Map interface in Java, which means multiple threads can access it simultaneously without any synchronization issues.
#### at this point we need to write the code that manage  cach entries by ourself
   - we need to write entry into our cache 
   - remove entry  from cache 
   - the most important thing we need to  remove the entry  from our cache if no one use this entry
   - clean our cache  peroidcally to reduce its size.

can we go further from this point? yes we can , welcome to caffeine

## caffeine
Caffeine is a high performance Java caching library providing a near optimal hit rate.

A Cache is similar to ConcurrentMap, but not quite the same. The most fundamental difference is that a ConcurrentMap persists all elements that are added to it until they are explicitly removed. A Cache on the other hand is generally configured to evict entries automatically, in order to constrain its memory footprint.
### Eviction types of Values
Caffeine has three strategies for value eviction: size-based, time-based, and reference-based.
- Size-Based Eviction
This type of eviction assumes that eviction occurs when the configured size limit of the cache is exceeded.

- Time-Based Eviction
This eviction strategy is based on the expiration time of the entry and has three types:
    - Expire after access — entry is expired after period is passed since the last read or write occurs
    - Expire after write — entry is expired after period is passed since the last write occurs
    - Custom policy — an expiration time is calculated for each entry individually by the Expiry implementation

- Reference-based
  We can configure our cache to allow garbage-collection of cache keys and/or values.  
  Caffeine allows you to set up your cache to allow the garbage collection of entries, by using weak references for keys or values, and by using soft references for values.  

### example  
```java
   Cache<Key, Graph> cache = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .maximumSize(10_000)
    .build();
   // Lookup an entry, or null if not found
   Graph graph = cache.getIfPresent(key);
   // Insert or update an entry
   cache.put(key, graph);
   // Remove an entry
   cache.invalidate(key);
```

 #implementation example  
 now with Buket4j and caffeine we can build an efficient Rate Limiter
 ```java
@Service
public class RateLimiter {
    //CaffeineProxyManager : is a class that provide an an integaraion between buket4j and caffeine
    
    @Autowired
    CaffeineProxyManager<String> caffeineProxyManager;

    public Bucket resolveBucket(String key) {
        Supplier<BucketConfiguration> configSupplier = getConfigSupplierForUser(key);
        
        //create buket is is not exist
        return caffeineProxyManager.builder().build(key, configSupplier);
    }

    private Supplier<BucketConfiguration> getConfigSupplierForUser(String userId) {
        //Buket Configuration
        Refill refill = Refill.intervally(5, Duration.ofMinutes(1));
        Bandwidth limit = Bandwidth.classic(5, refill);
        return () -> (BucketConfiguration.builder().addLimit(limit).build());
    }
}
//now we can use RateLimiter at any other classes by inject it
@RestController
public class CalculationController {
   
    @Autowired
    RateLimiter rateLimiter;
    
    @PostMapping(value = "/api/v1/area/rectangle")
    public ResponseEntity<AreaV1> rectangle(@RequestBody Dimensions dimensions) {
        logger.info(LocalDateTime.now().toString());
        Bucket bucket1 = rateLimiter.resolveBucket(dimensions.getId());
        if (bucket1.tryConsume(1)) {
            return ResponseEntity.ok(new Area("rectangle", dimensions.getLength() * dimensions.getWidth()));
        }
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS).build();
    }
 ```
