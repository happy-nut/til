# 12장. 리포지토리

에그리게잇에 대해서만 리포지토리를 제공해야 한다. 여기엔 일반적으로 1:1의 관계가 성립하는데, 때로 2개 이상의 애그리게잇이 계층 구조를 공유할 경우에는
그 타입들이 하나의 리포지토리를 공유할 수도 있다.

리토지토리를 설계하기 위해서는 2가지 설계 방법은 다음과 같다.

* 컬렉션 지향 리포지토리
* 영속성 지향 리포지토리

## 컬렉션 지향 리포지토리

데이터 저장소를 일종의 `HashSet`으로 바라보는 관점의 설계이다.

이를 위해선 영속성 메커니즘에서 암시적(implicit)으로 영속성 객체에 일어난 변화를 추적하는 기능이 지원되어야 한다.

다음은 변화 추적의 방식 중 일부다.

* 암시적 읽기 시 복사(Copy-on-Read): 읽어올 때 복사(지연 로딩 제외)하고 커밋할 때 원본과 비교. 변경이 감지된 모든 객체는 데이터 저장소에 반영한다.
* 암시적 쓰기 시 복사(Copy-on-Write): 모든 로드된 영속성 객체를 프록시를 통해 관리한다. 프록시는 변화 추적해서 객체에 더티 체크를 하고 데이터 저장소에 반영한다. 

아주 많은 객체를 가져와야 하며 매우 고성능의 시스템의 시스템을 설계해야 한다면 위와 같이 암시적으로 복사하고 변화를 추적하는 메커니즘은 불필요한 오버헤드일 수 있다.
이 경우 암시적 카피를 만드는 하이버네이트 대신에 카피를 만들지 않는 탑링크(Copy-before-Write 방식)를 영속화 메커니즘으로 고려해볼 수 있다. 

### 하이버네이트 구현

도메인 계층에 [인터페이스](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_identityaccess/src/main/java/com/saasovation/identityaccess/domain/model/identity/GroupRepository.java#L19) 를 정의한다.

코드를 보면, find 하는 메소드가 아니라면 전부 `void` 타입으로 반환하도록 되어 있다.

> 실무에선 새로 저장된 Entity의 아이디를 반환해주거나 해야하는 일이 있던데, 그 경우에는 어떡할까?

[실제 구현](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_identityaccess/src/main/java/com/saasovation/identityaccess/infrastructure/persistence/HibernateGroupRepository.java) 은 인프라 계층에 둔다.

하이버네이트의 HQL을 사용해서 따로 매퍼를 쓰지 않도록 했다. 아하.. 이러면 그냥 Pojo 를 써도 되는구나. 처음 알았다.

> 만약 `User` 애그리게잇안에 `Person` 애그리게잇이 있는데, `User`를 삭제하려고 한다면?
>
> * `Person` 먼저 삭제하고, `User`를 삭제한다.
>
> 위와 같이 1:1 연결은 실수가 발생할 수 있으므로, 일반적으로 1:1 연결 대신 제한된 1:N 연결을 사용한다.
> 
> CASCADE 처럼 A가 삭제되면 B도 삭제해! 라고 할 수도 있지만, 저자는 이 방식을 피한다. 영속성은 리파지토리로만 관리되어야 한다고 생각하기 때문이다.

### 탑링크 구현에 대한 고려

탑링크를 쓰게 되면 리포지토리의 추상화의 이점 없이 진행한다.

```java
Calendar calendar = session.readObject(...);

UnitOfWork unitOfWork = session.acquireUnitOfWork();

Calendar calendarToRename = unitOfWork.registerObject(calendar);

calendarToRename.rename("New Name");

unitOfWork.Commit();
```

`registerObject`가 '자! 너 나한테 이거 수정한다고 줬다? 명심해' 하고서 타깃의 클론을 만들어준다.

어찌뙜든 애그리게잇을 다시 저장소에 집어넣으려면 클론을 얻어야 하는데, 탑링크에서는 2가지 방법이 있다.

* 위 처럼 `registerObject` 사용
* `useEditingMode()` 를 통해 리포지토리를 편집하는 모드로 변경

> 번역이 똥같아서 못알아먹겠다. 옮긴이 양심 어딨냐 진짜루...

## 영속성 지향의 리포지토리

영속성 메커니즘이 암묵적으로나 명시적으로나 객체의 변화를 감지고하고 추적하지 못하는 상황이라면, 영속성 지향의 리포지토리를 사용해야 한다.
영속성 지향의 리포지토리를 사용하면 우리는 명시적으로 새롭고 변경된 객체를 저장소에 `put()` 해야 하는데, 이전에 주어진 키와 연관된 모든 값을 효과적으로 replace 해야 한다.

그 이유가 Unit Of Work 나 atomic write를 컨트롤할 수 있는 트랜잭션 경계를 지원하지 않기 때문이라는데, 뭔소린지 모르겠다. 하하

이 유형의 리포지토리는 그냥 캐시같은 걸 생각하면 될 것 같다. 

```java
cache.put(product.porductId(), product);

// 사용처
product = cache.get(productId);
```

### 코히어런스 구현

코히어런스는 데이터 패브릭에서 쓰이는 영속성 저장소를 말하는 것 같다.

```java
public interface ProductRepository {
  public ProductId nextIdentity();
  public Collection<Product> allProductsOfTenant(Tenant aTenant);
  public Product productOfId(Tenant aTenant, ProductId aProductId);
  public void remove(Product aProduct);
  public void removeAll(Collection<Product> aProductCollection);
  public void save(Product aProduct);
  public void saveAll(Collection<Product> aProductCollection);
}
```

영속성 지향 스타일에서는 애그리게잇이 생성될 때와 수정될 때 모두 저장되어야 한다(컬렉션 지향 스타일은 생성될 때만 저장된다).

```java
// 생성할 때
Product product = new Product(...);

productRepository.save(product); // 

// 수정할 때
Product product = productRepository.productOfId(tenantId, productId);

product.reprioiritizeFrom(backlogItemId, orderOfProority);

productRepository.save(product); // 수정될 때도 저장!
```

구현을 살펴보자. 책에서 특별히 언급한 부분은 주석으로 남겨놓았다.

```java
public class CoherenceProductRepository implements ProductRepository {
  private Map<Tenant, NamedCache> caches;
  
  public CoherenceProductRepsitory() {
    super();
    this.caches = new HashMap<Tenant, NamedCache>();
  }

  // cache()가 호출이 될 때가 되어서야 캐시를 얻어온다.
  private synchronized NamedCache cache(TenantId aTenantId) {
    NamedCache cache = this.caches.get(aTenantId);
    
    if (cache == null) {
      // 확장성을 얻기 위해 키 값에 prefix를 붙인다.
      cache = CacheFactory.getCache("ahilepm.Product." + aTenantId.id(), Product.class.getClassLoader());
      this.caches.put(aTenantId, cache);
    }
    
    return cache;
  }
  
  @Override
  public ProductId nextIdentity() {
    return new ProductId(UUID.randomUUID().toString().toUpperCase())
  }
  
  @Override
  public void save(Product aProduct) {
    this.cache(aProduct.tenantId()).put(this.idOf(aProduct), aProduct);
  }
  
  // 데이터 그리드에 영속화시킨다.
  @Override
  public void saveAll(Collection<Product> aProductCollection) {
    if (!aProductCollection.isEmpty()) {
      TenantId tenantId = null;
      
      Map<String, Product> productsMap = new HashMap<String, Product>(aProductCollection.saze());
      
      // 데이터 패브릭에서는 각 `put()`마다 네트워크 요청이 발생하기 때문에, 로컬 해쉬멥에 모아두고, 아래에서 `putAll()`로 단일 요청을 한다.
      for (Product product : aProductCollection) {
        if (tenantId == null) {
          tenantId = product.tenantId();
        }
        productMap.put(this.idOf(product), product);
      }
      
      this.cache(tenantId).putAll(productsMap);
    }
  }

  // removeAll의 구현. 왜 단일 네트워크 요청으로 하지 않을까? - 코히어런스가 해당 인터페이스를 제공하지 않는다.
  @Override
  public void removeAll(Collection<Product> aProductCollection) {
    for (Product product : aProductCollection) {
      this.remove(product);
    }
  }
  
  // find 메소드의 구현 - 코히어런스 필터 엔트리를 사용하는 대신, NamedCache 내의 모든 Product 인스턴스에 대한 요청을 하기만 하면 된다.
  @Override
  public Collection<Product> allProductsOfTenant(Tenant aTenant) {
    Set<Map.Entry<String, Product>> entries = this.cache(aTenant).entrySet();
    
    Collection<Product> products = new HashSet<Product>(entries.size());
    
    for (Map.Entry<String, Product> entry: entries) {
      products.add(entry.getValue());
    }
    
    return products;
  }
  ...
}
```

### 몽고DB 구현

아래 4가지 개괄적인 그림을 먼저 살펴보자.

1. 애그리게잇 인스턴스를 BSON 으로 직렬화한 후, 인스턴스를 가져올 때 역직렬화 한다.
2. 애그리게잇의 고유 식별자는 몽고DB에 의해 생성된다.
3. 몽고DB의 노드/클러스터로 참조할 수 있다.
4. 각 애그리게잇의 모든 인스턴스는 key-value pair로 저장되어야 한다.

구현을 살펴보자.

```java
public class MongoProductRepository extends MongoRepository<Product> implements ProductRepository {

  public MongoProductRepository() {
    super();
    // `BSONSerializer`는 역직렬화 시 필드를 매핑하여 지연 마이그레이션도 가능하도록 해준다.
    this.serializer(new BSONSerializer<Product>(Product.class));
  }
  
  public ProductId nextIdentity() {
    return new ProductId(new ObjectId().toString());
  }
  
  @Override
  public void save(Product aProduct) {
    this.databaseCollection(this.collectionName(aProduct.tenantId()))
      .save(this.serialize(aProduct));
  }
  
  @Override
  public Collection<Product> allProductsOfTenant(TenantId aTenantId) {
    Collection<Product> products = new ArrayList<Product>();
    
    DBCursor cursor = this.databaseCollection(this.databaseName(), this.collectionName(aTenantId)).find();
    
    while (cursor.haxNext()) {
      DBObject dbObject = cursor.next();
      
      Product product = this.deserialize(dbObject);
      
      products.add(product);
    }
    
    return products;
  }
  ...
}
```

## 추가적인 행동

리포지토리는 컬렉션을 가능한 한 많이 흉애내어야 하지만, 모든 인스턴스의 수를 세어주는 추가적인 행동의 이름은 `count`가 아니라 `size`가 적절하다. 만약
단순히 `select count(*) from ...` 로 구현되지 않고, 다른 추가적인 연산(비즈니스 로직을 수행하는 곳으로 데이터를 옮겨야 하는 등)이 필요한 경우라면
도메인 서비스를 이용하는 것이 좋다.

애그리게잇 루트에 접근하지 않고 리포지토리에서 애그리게잇의 부분을 쿼리하는 것은 어떨까? 애그리게잇이 일부 엔티티 타입의 큰 컬랙션을 가지고 있고
특정 기준에 부합하는 인스턴스에만 접근할 필요가 있을 때 그 방법을 적용해볼 수 있다. 그러나 편의를 위해 이렇게 설계하는 것은 최대한 지양하도록 하자.

다수의 리포지토리에 유스케이스 최적 쿼리를 지원하는 많은 `find` 메소드를 생성해야 한다면, 냄새 나는 코드일 확률이 높다.

* 리포지토리가 잘못된 애그리게잇 설계를 감추었다. 즉, 애그리게잇 경계를 잘못 판단해 하나 이상의 다른 타입의 애그리게잇을 설계해야 하는 기회를 놓쳤다.
* 애그리게잇이 잘 설계되었다면, CQRS를 고려해야할 시점일 수 있다.

## 트랜잭션의 관리

애플리케이션 계층에서 트랜잭션을 관리한다. 일반적으로 애플리케이션에서 처리할 각 주요 유스케이스 그룹당 하나의 파사드를 애플리케이션 계층에 생성한다.

코드로 보자면 다음과 같다.

```java
public class SomeApplicationServiceFacade {
  ...
  public void doSomeUseCaseTask() {
    Transaction transaction = null
    
    try {
      transaction = this.session().beginTransaction();
      // 도메인 모델 사용...
      transaction.commit();    
    } catch (Exception e) {
      if (transaction != null) {
        transaction.rollback();
      }
    }
  }
}
```

하이버네이트는 이를 어노테이션 하나로 간편히 작업할 수 있게 해준다.

```java
public class SomeApplicationServiceFacade {
  ...
  @Transactional
  public void doSomeUseCaseTask() {
    // 도메인 모델 사용...
  }
}
```

하이버네이트 뿐만 아니라 어떤 영속성 메커니즘이든지 같은 session, Unit of work 그리고 트랜잭션으로의 접근을 제공할 방법을 반드시 찾아야 한다.

### 경고

애그리게잇은 올바른 일관성 경계를 확보하기 위해 신중하게 설계되어야 한다.
만약 신중하게 설계되지 않아 단일 트랜잭션에서 여러 애그리게잇의 수정을 커밋하는 기능을 과도하게 구현하게 되면 동시성 문제로 겪을 수 있다.

## 타입 계층 구조

도메인 모델끼리 타입 계층구조를 가지게 되는 경우엔 어떨까?

`FooServiceProvider` 와 `BarServiceProvider` 를 범용적으로 스케줄링하기 위해 다음과 같이 구현했다고 치자.

```java
serviceProviderRepository.providerOf(id).scheduleService(date, description);
```

이 경우 `FooServiceProvider`와 `BarServiceProvider`를 반환하지 않고, 부모인 `ServiceProvider`를 반환한다.

그러나 LSP를 위반하여 서브 클래스를 반환하는 경우라면 클라이언트는 다음과 같은 코드를 작성하게 된다.

```java
if (id.identifiesFoo()) {
  serviceProviderRepostory.FooOf(id).scheduleFooService(date, fooDescription);
} else if (id.identifiesBar()) {
  serviceProviderRepostory.BarOf(id).scheduleBarService(date, barDescription);
}
```

이건 냄새나는 코드다. 구체적인 서브 클래스 수가 2개 정도라면 리포지토리를 2벌 준비하는 게 낫고, 서브클래스 수가 그것을 초과하고 대부분이
완전히 상호 교체가능하도록 사용될 수 있다면(LSP) 공통의 리포지토리를 공유하는 편이 더 낫다.

대부분의 경우에는 타입을 설명하는 정보를 애그리게잇 속성으로 설계하면 이런 상황을 피할 수 있다.

```java
public class ServiceProvider {
  private ServiceType type;
  
  public void scheduleService(Date aDate, ServiceDescription aDescription) {
    if (type.isFoo()) {
      this.scheduleFooService(aDate, aDescription);
    } else if (type.isBar()) {
    }this.scheduleBarService(aDate, aDescription);
  }
}
```

내부의 if 문들이 지저분하다면 더 작은 계층 구조를 또 하나 설계해서 처리하도록 하면 된다.

역할 기반의 인터페이스를 사용해보는 방법도 생각해볼 수 있다. 여러 애그리게잇 타입이 `SchedulelableService` 인터페이스를 구현하는 것이다.

## 리포지토리 대 DAO

DAO와 관련된 패턴을 데이터베이스 테이블의 wrapper로 적용되는 경향이 있지만, 리포지토리는 도메인 모델과 데이터 매퍼가 전형적으로 함께 사용된다는 점에서
둘은 서로 다르다. 둘을 동급으로 취급하지 말고 리포지토리를 최대한 컬렉션 지향으로 설계 해야 우리는 도메인 모델에 집중할 수 있게 된다.

## 리포지토리의 테스트

2가지 방향이 있다.

1. 리포지토리 자체가 바르게 동작하는지 테스트: 프로덕션 수준 품질의 구현을 사용해야 한다.
2. 리포지토리의 사용처를 테스트: 인메모리 구현을 사용할 수도 있다.

첫번째 경우부터 보자.

`setUp()`과 `tearDown()` 으로 각 테스트 후 수행해야 하는 오퍼레이션을 구현한다. 그리고 실제로 IO를 수행하면서 실제로 리포지토리가 동작하는지 테스트한다.

### 인메모리 구현으로 테스트하기

클라이언트를 테스트하기 위해 리포지토리 전체 영송석 구현을 설정하는 일이 매우 어렵거나 사용하기에 너무 느리다면 `HashMap` 같은 것을 통해 인메모리 구현을 이용할 수 있다.
