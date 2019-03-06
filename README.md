# Spring Security ACL With Redis Cache
## Project Structure
기존 프로젝트는 Java 7 버전을 사용중이며 Spring 4.0.4.RELEASE 버전을 쓰고 있었다.

권한과 관련하여 Spring Security ACL을 사용하며 AclCache를 위해 JBoss의 JNDI로 설정 된   
nativeCacche를 SpringEmbeddedCacheManager에 주입하여 사용하고 있었다.  

즉, LocalCache를 사용하고 있는 구조이며 그 설정부분을 간추리면 아래와 같다.

> cache configuration 파일
```xml
<cache:annotation-driven/>

<jee:jndi-lookup id="nativeCacheManager"
                 expected-type="org.infinispan.manager.EmbeddedCacheManager"
                 jndi-name="java:jboss/infinispan/container/application"/>

<bean id="cacheManager" class="org.infinispan.spring.provider.SpringEmbeddedCacheManager">
    <constructor-arg ref="nativeCacheManager"/>
</bean>
```

> JBoss domain.xml 파일
```xml
<subsystem xmlns="urn:jboss:domain:infinispan:1.5">
    <cache-container name="web" aliases="standard-session-cache" default-cache="local-web" 
                module="org.jboss.as.clustering.web.infinispan" statistics-enabled="true">
        <local-cache name="local-web" batching="true" statistics-enabled="true">
            <file-store passivation="false" purge="false"/>
        </local-cache>
    </cache-container>
    <cache-container name="hibernate" default-cache="local-query" 
                module="org.jboss.as.jpa.hibernate:4" statistics-enabled="true">
        <local-cache name="entity" statistics-enabled="true">
            <transaction mode="NON_XA"/>
            <eviction strategy="LRU" max-entries="10000"/>
            <expiration max-idle="100000"/>
        </local-cache>
        <local-cache name="local-query" statistics-enabled="true">
            <transaction mode="NONE"/>
            <eviction strategy="LRU" max-entries="10000"/>
            <expiration max-idle="100000"/>
        </local-cache>
        <local-cache name="timestamps" statistics-enabled="true">
            <transaction mode="NONE"/>
            <eviction strategy="NONE"/>
        </local-cache>
    </cache-container>
    <cache-container name="application" default-cache="local-acl" start="EAGER" 
                    module="org.jboss.as.clustering.web.infinispan" statistics-enabled="true">
        <local-cache name="local-acl" statistics-enabled="true">
            <transaction mode="NONE"/>
            <eviction strategy="LRU" max-entries="10000"/>
        </local-cache>
    </cache-container>
</subsystem>
```

## 공유 캐시 설정의 필요성
현재 프로젝트에선 Applicaiton 구동 된 후 런타임에 권한 정보를 DB에서 읽어와 AclCache에 담는데   
Role에 대한 Privilege가 바뀔 경우를 대비하여 AchCache를 clear하도록 구현이 필요했다.  
(내가 이 프로젝트를 맡기 이전엔 캐시 clear 하는 방법을 몰라 WAS를 내렸다 올렸다고 한다)
  
변경 된 Role에 대한 권한 정보만 삭제하면 되지 않을까 했지만 Spring Security ACL에서 AclCache가 저장되는 구조는  
"어떤 Entity에 대해 어떤 Object, Role이 어떤 Permission을 가지고 있다" 라는 형태로 정의되어있다.

따라서 저장 된 Cache의 full-scan을 통해 조회하여 삭제해야했고 이러한 구조적 문제로 AclCache 인터페이스에서도  
clearCache 메소드만 제공하고 있었다. 
```java
public interface AclCache {
    //~ Methods ============================================================================
    
    void evictFromCache(Serializable pk);

    void evictFromCache(ObjectIdentity objectIdentity);

    MutableAcl getFromCache(ObjectIdentity objectIdentity);

    MutableAcl getFromCache(Serializable pk);

    void putInCache(MutableAcl acl);

    void clearCache();
}
```

이때 발생하는 문제가 이중화 구성된 서버일 경우 Cache의 클러스터링 설정이 반드시 필요하다는 것인데  
JBoss의 버전 문제인지 클러스터링이 제대로 동작하지 않았다.  

이로인해 아래와 같은 문제가 지속적으로 발생했다  
> 이중화서버 A에서 ADMIN role에 모든 읽기 쓰기 권한을 해제하여 저장  
> A 서버에는 기존 권한 캐시가 비워지며 새로운 권한이 조회 되고 이것이 캐시에 새로 저장 됨  
> B 서버에는 기존 권한 캐시가 그대로 남아있음  
> LB 설정이 Round Robin으로 되어있어 A 서버에 붙는 사용자와 B 서버에 붙는 사용자가 서로 다른 화면을 보게 됨  
> 문제 발생 시 마다 매뉴얼적으로 B 서버의 캐시를 비우도록 임시 대응  

엔지니어께서 Cache clustering 설정을 적용하고 테스트하는데에 여력이 없어 Redis를 통해 AclCache를  
공유캐시로 설정하도록 결정하였다.  

## Redis 설정  
이전에 Spring에 Redis Cache를 연동해본 경험이 있어 개발 환경에 Window 용 Redis-3.2.100이 설치되어있었다.  
Spring의 버전을 고려해 아래와 같이 Spring-data-redis와 Redis client인 Jedis 등의 의존을 추가했다.
  
```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>1.7.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.7.0</version>
</dependency>
```

그 후 아래와 같이 기존에 nativeCacheManager를 사용하는 것을 RedisCacheManager로 변경하고  
Spring Data Redis의 RestTemplate을 주입해주었다.  
```xml
<bean id="jedisConnectionFactory" 
    class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
    <property name="hostName" value="127.0.0.1" />
    <property name="port" value="6379" />
    <property name="usePool" value="true" />
</bean>

<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="jedisConnectionFactory" />
</bean>

<bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
    <constructor-arg ref="redisTemplate" />
</bean>
```

정상적으로 캐시가 쌓이는 것을 확인했으나 ClearCache가 제대로 동작하지 않았다.  
jedis에선 flushAll, flushDB 등의 캐시를 날리는 메소드를 지원하였으나   
RestTemplate에서는 해당 메소드를 expose하지 않고 있었다.  

그래서 아래와 같이 RedisTemplate의 callback을 통해 jedis의 flushAll을 호출하는  
RedisCache 구현체를 만들고 이 구현체를 Managing 하도록 하는 CustomRedisCacheManager를 구현했다  
```java
public class FlushRedisCache extends RedisCache {
    private static final Logger log = LoggerFactory.getLogger(AsyncFlushRedisCache.class);
    private final RedisOperations operations;
    
    public FlushRedisCache(String name, byte[] prefix, 
                                RedisOperations<?, ?> template, long expiration) {
        super(name, prefix, template, expiration);
        this.operations = template;
    }

    @Override
    public void clear() {
        try {
            this.operations.execute(new RedisCallback() {
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    connection.flushAll();
                    log.info("flushAll action succeed");
                    return null;
                }
            });
            
        } catch (Exception e) {
            log.warn("모든 캐시를 삭제하는데 실패했습니다.", e);
        }
    }
}
```

```java
public class FlushRedisCacheManager extends RedisCacheManager {
    public AsyncFlushRedisCacheManager(RedisTemplate template) {
        super(template);
    }

    @Override
    protected RedisCache createCache(String cacheName) {
        long expiration = computeExpiration(cacheName);
        return new FlushRedisCache(cacheName, 
        (isUsePrefix() ? getCachePrefix().prefix(cacheName) : null), getRedisOperations(), expiration);
    }
}
```

Cache configuration xml에 위에 구현한 Cachemanager를 등록하고 테스트한 결과는 정상적으로 캐시가 비워졌었다.  
그런데 Redis 관련 레퍼런스를 찾다가 flushAll, flushDB, scan 등 명령은 Single Thread 구조 인 Redis에서  
병목지점을 만들어 낼 수 있다는 것을 봤다.  
결국 운영중에 권한이 변경 될 때마다 자주 캐시가 비워져야하는 해당 Application에선 장애가 날 수 있다는 것.  

Single Thread 구조가 아닌 MemCached를 사용하거나 캐시 클러스터링 설정이 제대로 동작하도록 바라는 것에  
의존을 해야하는 상황이었다.  

다행히 Redis 4.0 이상 버전부터 지원하는 FLUSHALL ASYNC 명령을 사용하면 이러한 문제점을 개선할 수 있다고 한다.  

## Redis 4.0 설정
먼저 Local에 설치한 Window 전용 Redis는 3.2.100 버전이므로 4.0을 설치하기 위해 구글링을 하였으나  
Window용 Redis는 해당 버전을 마지막으로 지원이 중단 된 상태였다.  
우분투를 윈도우에 설치 후 Redis 설치하는 방법이 있는 것으로 보았는데 여기서는 Docker를 이용하기로 했다. 
  
docker run --name some-redis -d -p 6379:6379 redis 명령으로 Redis를 컨테이너에 올려 설치는 완료.  

FLUSHALL ASYNC 명령을 날리기 위해 작성한 callback을 아래와 같이 직접 커맨드를 날리도록 수정했으나  
Jedis의 Command가 이를 지원하지 않았다. 디버깅을 통해 따라가보니 FLUSHALL, FLUSHDB 등은 있어도  
ASYNC와 관련된 명령어는 없었다. 

```java
public class FlushRedisCache extends RedisCache {
    private static final Logger log = LoggerFactory.getLogger(AsyncFlushRedisCache.class);
    private final RedisOperations operations;
    
    public FlushRedisCache(String name, byte[] prefix, 
                                RedisOperations<?, ?> template, long expiration) {
        super(name, prefix, template, expiration);
        this.operations = template;
    }

    @Override
    public void clear() {
        try {
            this.operations.execute(new RedisCallback() {
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    connection.execute("flushall async");
                    log.info("flushAll action succeed");
                    return null;
                }
            });
            
        } catch (Exception e) {
            log.warn("모든 캐시를 삭제하는데 실패했습니다.", e);
        }
    }
}
```

Jedis의 버전이 낮아 지원하지 않는 것으로 파악하였는데 Spring Session의 default redis client가 Lettuce로  
바뀐 것과 NIO에 대한 지원 등이 Lettuce가 더 나은 대처를 보인 것 같아 Redis client를 바꾸기로 했다.  

## Lettuce와 Spring Data Redis
<a href="https://github.com/lettuce-io/lettuce-core/issues/146">Lettuce의 Github</a>에서 2015년 10월 경 FLUSHALL ASYNC에   
대한 지원이 적용된 이슈를 확인했다.   
또한 반영된 커밋에서 flushallAsync 메소드가 인터페이스에 선언 된 것을 확인했다.  
<img src="https://raw.githubusercontent.com/dlxotn216/spring-security-acl-with-redis-cache/master/images/flushallasync.png" />

Maven repository에서 Lettuce redis client를 찾았는데 3.x, 4.x, 5.x의 group id가 다 달랐다.  
일단 Lettuce의 최신 버전인 lettuce-core 5.0.1을 적용해보았으나 역시나 제대로 동작하지 않았다.  
4.2 버전 이상부터는 Java8을 요구하는데 현재 프로젝트는 java7 기반이므로 문제가 있었다.  

따라서 사용가능한 Lettuce 버전은 3.x이다. 여기서 사전 확인이 필요한 것이 3.x에 과연 flushallAsync에 대한 지원이  
있는 것인가이다. <a href="https://github.com/lettuce-io/lettuce-core/tree/3.5.x">3.5.x 브랜치</a>를 통해 하나하나 소스파일을 찾아보려 했으나 너무 비효율적이었다.  

다행히도 <a href="https://lettuce.io/lettuce-3/release/api/">Letucce 3.x에 대한 Javasoc</a>을 찾았고 index 탭에서 flushallAsync 메소드를 찾았다.  
<img src="https://raw.githubusercontent.com/dlxotn216/spring-security-acl-with-redis-cache/master/images/flushallasyncInJavaDoc.png" />
따라서 com.labmdaworks의 Lettuce-core 3.5.0.final을 pom.xml에 추가하였다  

이것에 맞추어 Spring Data Redis의 버전도 바꾸기로 했다. <a href="https://docs.spring.io/spring-data-redis/docs/current/reference/html/#new-in-1.7.0">Spring Data Redis docs</a>의 릴리즈 노트를 보면  
1.8부터 Jedis 2.9, Lettuce 4.2(Required Java 8)이 명시되어있다. 따라서 여기에선 1.7 버전을 적용했다.  

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>1.7.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>biz.paluch.redis</groupId>
    <artifactId>lettuce</artifactId>
    <version>3.5.0.Final</version>
</dependency>
```

모든 의존 설정을 마쳤으니 마지막으로 JedisConnectionFactory를 LettuceConnectionFactory로 바꾸어 Spring을 띄웠다. 
 
```xml
<bean id="lettuceConnectionFactory" c
        lass="org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory">
    <property name="hostName" value="192.168.99.100" />
    <property name="port" value="6379" />
</bean>    

<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="lettuceConnectionFactory" />
</bean>

<bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
    <constructor-arg ref="redisTemplate" />
</bean>
```
(usePool 옵션이 사라졌다. Lettuce는 Redis와는 달리 Netty 라이브러리 위에 구축되었고 Connection 인스턴스를  
여러 스레드에서 공유가 가능하기 때문에 jedis-pool과 같은 설정은 필요 없다)

정상적으로 Spring이 뜨고 Application이 구동되었으니 이제 RedisCallback 부분에서 flushallAsync 메소드를 이용하면 된다.  
하지만 역시나 문제가 생겼다. 바로 flushallAsync 메소드가 안보인다...  
<img src="https://raw.githubusercontent.com/dlxotn216/spring-security-acl-with-redis-cache/master/images/whereisflushallasync.png" />

다시 Javadoc을 보면 RedisConnection 인터페이스의 flushallAsync 메소드는 RedisServerConnection 인터페이스로부터  
상속 된 것이라고 나와잇다.  

> **Methods inherited from interface com.lambdaworks.redis.RedisServerConnection**  
bgrewriteaof, bgsave, clientGetname, clientKill, clientKill, clientList, clientPause, clientSetname, command, 
commandCount, commandInfo, commandInfo, configGet, configResetstat, configRewrite, configSet, dbsize, debugHtstats, 
debugObject, debugOom, debugSegfault, flushall, **flushallAsync**, flushdb, flushdbAsync, info, info, lastsave, save, 
shutdown, slaveof, slaveofNoOne, slowlogGet, slowlogGet, slowlogLen, slowlogReset, sync, time  

근데 왜 Callback에서 받은 RedisConnection에는 해당 메소드가 안보일까?...  
정답은 간단하다 Callback의 RedisConnection은 Spring Data Redis에서 추상화한 RedisConnection 인터페이스이다.  
즉 org.springframework.data.redis.connection.RedisConnection에는 flushallAsync 메소드가 없음을 의미한다.  

그렇다면 여기서 필요로하는  com.lambdaworks.redis.RedisConnection 인터페이스 타입의 커넥션은 어떻게 가져올까?  
정답은 생각보다 간단했다. org.springframework.data.redis.connection.RedisConnection의 getNativeConnection() 메소드를  
호출하면 된다. 아래와 같이 디버거를 통해서도 확인하면 RedisAsyncConnectionImpl 타입인것을 확인할 수 있다.    
<img src="https://raw.githubusercontent.com/dlxotn216/spring-security-acl-with-redis-cache/master/images/getNativeConnection.png" />

최종적인 구현은 아래와 같다. 혹시 모를 형 변환오류에 대비하여 instanceof를 통해 타입 비교를 하였고  
예외의 상황을 대비하여 flushallAsync 메소드 호출이 불가능하다면 flushAll 메소드를 통해 예외의 상황에서도  
캐시가 비워지도록 처리했다.  

또한 커스텀 구현체의 Naming을 FlushRedisCache에서 AsyncFlushRedisCache로,   
FlushRedisCacheManager에서 AsyncFlushRedisCacheManager로 바꾸었다. 
```java
public class AsyncFlushRedisCache extends RedisCache {
    private static final Logger log = LoggerFactory.getLogger(AsyncFlushRedisCache.class);
    private final RedisOperations operations;
    
    public AsyncFlushRedisCache(String name, byte[] prefix, RedisOperations<?, ?> template, long expiration) {
        super(name, prefix, template, expiration);
        this.operations = template;
    }

    @Override
    public void clear() {
        try {
            this.operations.execute(new RedisCallback() {
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    if(connection.getNativeConnection() instanceof RedisAsyncConnection){
                        ((RedisAsyncConnection) connection.getNativeConnection()).flushallAsync();
                    } else {
                        connection.flushAll();
                    }
                    log.info("flushAll action succeed");
                    return null;
                }
            });
            
        } catch (Exception e) {
            log.warn("모든 캐시를 삭제하는데 실패했습니다.", e);
        }
    }
}
```

## 마무리
사실 이번 이슈는 Spring, Java, Redis 등등 버전의 호환성을 찾아다니느라 시간이 오래걸렸다.  
특히나 Spring data redis 버전이 1.x에서 2.x로 넘어감에 따라 요구하는 Lettuce-core의 버전이 달라져  
LettuceConnectionFactory 클래스가 온통 빨간불이 들어온 문제도 있었고  
RedisTemplate이 아닌 RedisOperation을 사용하도록 메소드 시그니처가 변경되어 
AsyncFlushRedisCache나 AsyncFlushRedisCacheManager 클래스에 온통 빨간불이 들어온 문제가 있었다. 

java 버전에 맞는, flushallAsync 메소드를 지원하는 Redis client 버전을 찾고  
또 그 버전과 호환되는 Spring Data Redis의 버전을 찾는것이 꽤나 번거로웠다.  
 
이래서 Spring Boot를 사용하는구나 싶었고   
Java나 Spring에 대한 Version 마이그레이션을 미리미리 진행하는게 어떨까 싶다.  
Java7 -> Java8에 대한 마이그레이션은 큰 문제도 없을 것일 뿐더러  
Spring 4.0.4 -> 4.3.26(final)도 마찬가지라고 생각한다.


