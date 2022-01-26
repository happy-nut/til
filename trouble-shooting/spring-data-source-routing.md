# Spring 에서 Data source routing 하기

### 문제 정의

다음과 같이 data source의 라우팅 목적지가 3 군데로 나뉘게 되었다.

```kotlin
enum class DataSourceRoute {
    RW,
    RO,
    RO3
}
```

쓰기 요청이라면 `RW` data source 와 연결하고, 읽기 요청이라면 `RO` data source 와 연결하고, Batch job이 대용량 쿼리를 날릴 땐 `RO3` data source와 연결이 되어야 한다. 하지만 이렇게 데이터 소스를 라우팅하는 것은 비즈니스 로직과는 무관한 횡단 관심사(cross-cuttng-concerns) 다. AOP 를 통해 구조를 깔끔하게 가져가볼 수 있지 않을까.

### 문제 해결

#### AbstractRoutingDataSource

스프링에서는 [`AbstractRoutingDataSource`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.html) 라는 추상 클래스를 제공해준다. 이 추상 클래스를 구현하면 아래처럼 key 기반으로 런타임에 동적으로 연결할 data source를 결정할 수 있다.

```kotlin
class ReplicationRoutingDataSource : AbstractRoutingDataSource() {
    override fun determineCurrentLookupKey(): Any {
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            return DataSourceRoute.READ
        }

        return DataSourceRoute.WRITE
    }
}
```

물론 key 를 가지고 있는 자료 구조는 아래에서 처럼 `LazyConnectionDataSourceProxy` 를 통해 라우팅이 될 수 있도록 설정되어 있어야 한다.

```kotlin
@Bean
fun routingDataSource(): DataSource {
    val routingDataSource = ReplicationRoutingDataSource()
    val dataSourceMap = mapOf<Any, Any>(
        DataSourceRoute.RW to writeDataSource(),
        DataSourceRoute.RO to readDataSource(),
        DataSourceRoute.RO3 to batchDataSource()
    )
    routingDataSource.setTargetDataSources(dataSourceMap)
    routingDataSource.setDefaultTargetDataSource(dataSourceMap[DataSourceRoute.RW]!!)
    return routingDataSource
}

@Bean("dataSource")
fun dataSource(): DataSource {
    return LazyConnectionDataSourceProxy(routingDataSource())
}
```

위와 같은 방식 만으로도 `Transactional(readOnly = true)` 면 `READ`로 라우팅되도록, 그렇지 않다면 `WRITE`로 라우팅되도록 할 수 있지만, 문제는 여기서 `BATCH` 라는 라우트 타입이 추가되면서 `isCurrentTransactionReadOnly` 만으로는 판단이 힘들어졌다는 것이다.

#### ThreadLocal 에 route 를 저장

`TransactionManager`의 연결이 호출될 때마다 위 `determineCurrentLookupKey()`가 불려지는데, 이 때 찾아진 data source connection은 현재 스레드에 바인딩된다.

```kotlin
object DataSourceContext {
    val route = ThreadLocal.withInitial { DataSourceRoute.RW }!!
}
```

결국 트랜잭션이 스레드마다 바인딩된다는 것이므로 위처럼 `ThreadLocal` 변수를 사용하여 각 트랜잭션 마다 경로를 지정해 볼 수 있을 것 같다.

#### AbstractRoutingDataSource 수정

`DataSourceContext`의 스레드 로컬 변수에 설정된 경로를 읽어 연결할 경로를 찾도록 하기 위해 다음과 같이 `AbstractRoutingDataSource` 를 변경한다.

```kotlin
class ReplicationRoutingDataSource : AbstractRoutingDataSource() {
    override fun determineCurrentLookupKey(): Any {
        println("${Thread.currentThread().name} routed to ${DataSourceContext.route.get()}")
        return DataSourceContext.route.get()
    }
}
```

#### Annotation 구현

우선 어노테이션을 다음과 같이 선언하여, Repository의 각 메소드가 어떤 data source로 향할지 설정할 수 있도록 한다.

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.TYPE)
@Retention(AnnotationRetention.RUNTIME)
annotation class RoutingDataSource(
    val value: DataSourceRoute = DataSourceRoute.RW
)
```

어노테이션 구현체는 `ThreadLocal` 변수를 세팅하고, 트랜잭션이 다 끝났으면 다시 제거하도록 구현한다.

```kotlin
@Aspect
@Component
class RoutingDataSourceInterceptor {
    @Around("@annotation(routingDataSource)")
    fun routingDataSource(
        joinPoint: ProceedingJoinPoint,
        routingDataSource: RoutingDataSource
    ): Any {
        return runCatching {
            DataSourceContext.route.set(routingDataSource.value)
            joinPoint.proceed()
        }.also {
            DataSourceContext.route.remove()
        }
    }
}
```

#### 검증

테스트를 위해 다음과 같이 레포지토리를 작업해놓는다.

```kotlin
@RoutingDataSource
@Transactional
fun testRw() {
    testRepository.findById(1)
}

@RoutingDataSource(DataSourceRoute.RO)
@Transactional
fun testRo() {
    testRepository.findById(1)
}

@RoutingDataSource(DataSourceRoute.RO3)
@Transactional
fun testRo3() {
    testRepository.findById(1)
}
```

각 경로가 해당 트랜잭션을 테스트 할 수 있도록 하고, 다음과 같이 테스트 요청을 날려보았다.

```
❯ curl -X GET "http://localhost:8080/test-ro"
❯ curl -X GET "http://localhost:8080/test-rw"
❯ curl -X GET "http://localhost:8080/test-ro3"
❯ curl -X GET "http://localhost:8080/test-ro"
❯ curl -X GET "http://localhost:8080/test-rw"
❯ curl -X GET "http://localhost:8080/test-ro3"
❯ curl -X GET "http://localhost:8080/test-ro3"
```

결과:

```
http-nio-8080-exec-1 routed to RO
http-nio-8080-exec-2 routed to RW
http-nio-8080-exec-4 routed to RO3
http-nio-8080-exec-6 routed to RO
http-nio-8080-exec-8 routed to RW
http-nio-8080-exec-10 routed to RO3
http-nio-8080-exec-2 routed to RO3
```

#### Annotation Order

내가 만든 `RoutingDataSource` 는 `Transactional` 어노테이션보다 우선순위가 낮아야 한다. `Transactional` 어노테이션의 프록시를 타기에 앞서 목적지 data source가 결정되어야 하기 때문이다. 따라서 Configuration 에 다음 코드를 추가해 `Transactional` 어노테이션이 항상 안쪽 프록시에서 실행될 수 있도록 한다.

```kotlin
@EnableTransactionManagement(order = Ordered.HIGHEST_PRECEDENCE)
```

문제 없이 잘 실행된다.&#x20;
