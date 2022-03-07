# 7장. 서비스

## 도메인 서비스란 무엇인가

도메인 서비스란 도메인 고유의 작업을 수행하는 **무상태**의 오퍼레이션이다. 주로 애그리게잇이나 값 객체 상에서 수행해야 하는 오처레이션이 메소드로는 부적절하게 느껴질 때 사용한다.

애그리게잇에서 repository를 불러야만 행동의 구현이 가능하다 = 도메인 서비스로 빼자!

> 정말 애그리게잇의 행동이라면 정적 메소드로 선언하는 것은 이상할까? - 비즈니스 우선 순위를 고려할 때 그게 정말 맞는 방법일까?

도메인 서비스에 서비스라는 단어가 들어있다고 해서 엄청 큰 단위 집합체이거나 원격 기능이 있고 무거운 트랜잭션 오퍼레이션이라는 의미는 아니다.
물론 외부 바운디드 컨텍스트 상의 호출과 관련이 있는 경우도 있지만, 도메인 서비스 자체가 RPC 인터페이스를 제공하진 않고, 오히려 RPC를 사용하는 클라이언트가 된다.

또, 도메인 서비스를 애플리케이션 서비스와 혼동해선 안된다. 애플리케이션 서비스는 도메인 모델의 자연스러운 클라이언트로서 보통 도메인 서비스의 클라이언트가 된다.

도메인 서비스는 다음의 경우에 사용할 수 있다.

* 중요한 비즈니스 프로세스를 수행할 때
* 어떤 컴포지션에서 다른 컴포지션으로 도메인 객체를 변형할 때
* 하나 이상의 도메인 객체에서 필요로 하는 입력 값을 계산할 때

도메인 서비스는 무상태이며, 유비쿼터스 언어를 확실히 표현하는 인터페이스를 갖고 있음을 분명히 하자.

## 서비스가 필요한지 확인하자

도메인 서비스는 상황이 적절할 때만 사용해야 한다. 서비스를 지나치게 사용하면 서비스에만 비즈니스 로직이 몰려 에너믹 모델이 만들어진다.

도메인 서비스의 도입이 필요한 다음 요구 사항을 보자.

* 시스템의 사용자는 반드시 인증돼야 하지만, 테넌트가 활성화된 경우에만 인증이 가능하다.
 
만약 `User` 에 이 행동을 두면 어떻게 될까?

```java
boolean authentic = false;

User user = DomainRegistry.userRepository().userWithUsername(aTenantId, aUsername);

if (user != null) {
  authentic = user.isAuthentic(aPassword);
}

return authentic;
```

위 코드는 다음 문제점들이 있다.

* 클라이언트가 인증하려면 `User` 를 찾아 `User`에게 비밀번호가 일치하는 지 물어봐야 한다.
* 같은 맥락에서, 유비쿼터스 언어가 명시적으로 모델링 되지 않았다. `Tenant`가 활성화되어야 인증이 가능하지 않았던가?

그럼 `Tenant`로 인증 로직을 옮겨보자.

```java
boolean authentic = false;

Tenant tenant = DomainRegistry.tenantRepository().tenantOfId(aTenantId);

if (tenant != null && tenant.isActive()) {
  User user = DomainRegistry.userRepository().userWithUsername(aTenantId, aUsername);
  if (user != null) {
    authentic = tenant.authenticate(user, aPassword);
  }
}

return authentic;
```

여전히 문제점이 있다.

* 클라이언트는 이제 인증 절차까지 알아야 해서 부담이 더 커졌다.

이런 식으로 어디에 두어도 어색할 때, 우리는 서비스가 필요하다고 느낄 수 있다.

```java
UserDescriptor userDescriptor = DomainRegistry.authenticationService().authenticate(aTenantId, aUsername, aPassword);
```

`UserDescriptor`는 `User`를 참조하기 위해 필수적인 일부 특성만을 포함한 값 객체이고 사용자별 세션을 다룬다.

## 도메인 서비스를 모델링하기

분리된 인터페이스가 있어야 한다면, 도메인 모듈에 다음과 같이 선언해볼 수 있다.

```java
// domain 패키지
public interface AuthenticationService {
  public UserDescriptor authenticate(TenantId aTenantId, String aUsername, String aPassword);
}
```

기술적 세부사항을 모르도록 하고 싶다면 인프라에 내린다.

```java
// infra 패키지
public class DefaultEncryptionAuthenticationService implements AuthenticationService {
  public DefaultEncryptionAuthenticationService() {
    super();
  }
  
  @Override
  public UserDescriptor authenticate(TenantId aTenantId, String aUsername, String aPassword) {
    if (aTenantId == null) {
      throw new IllegalArgumentException("TenantId must not be null");
    }
    
    if (aUsername == null) {
      throw new IllegalArgumentException("Username must not be null");
    }
    
    if (aPassword == null) {
      throw new IllegalArgumentException("Password must not be null");
    }
    
    UserDescriptor userDescriptor = null;
    
    Tenant tenant = DomainRegistry.tenantRepository().tenantOfId(aTenantId);
    
    if (tenant != null && tenant.isActive()) {
      String encryptedPassword = DomainRegistry.encryption
    }
  }
} 
```

### 분리된 인터페이스가 꼭 필요할까

주어진 여러 `Tenant`가 특별한 보안 표준을 요구할 수 있다는 측면에서 인프라로 빼는 것도 괜찮지만, 지금 상황에서는 다음과 같이 구현하는 것도 괜찮다.

```java
// 도메인 패키지
public class AuthenticationService {
  public AuthenticationService() {
    super();
  }
  
  public UserDescriptor authenticate(TenantId aTenantId, String aUsername, String aPassword) {
    // 위와 동일한 코드.
  }
}
```

분리된 인터페이스는 결합 분리의 목표가 분명할 때 유용하다: 인터페이스로의 의존성이 필요한 클라이언트는 해당 구현에 관해서 전혀 몰라도 된다.

그러나 목표가 분명치 않더라도, 의존성 주입이나 서비스 팩토리를 사용한다면 같은 효과를 얻을 수 있다.

```java
// 서비스 팩토리를 사용한다면:
UserDescriptor userDescriptor = DomainRegistry.authenticationService().authenticate(aTenantId, aUsername, aPassword);
```

```java
// 의존성 주입을 사용한다면:
public class SomeApplicationService ...{
  @Autowired
  private AuthenticationService authenticationService;
  ...
}
```

이 밖에, 생성자를 통해 의존성을 설정하거나 매개변수로 직접 전달해도 상관은 없다. 테스트 편이성 등을 고려해 알아서 잘 결정해보자.

### 계산 프로세스

코드 안의 주석을 보자.

```java
public class BusinessPriorityCalculator {

  private BacklogItemRepository backlogItemRepository;

  public BusinessPriorityCalculator(BacklogItemRepository aBacklogItemRepository) {
    super();
    this.backlogItemRepository = aBacklogItemRepository;
  }

  public BusinessPriorityTotals businessPriorityTotals(TenantId aTenantId, ProductId aProductId) {
    int totalBenefit = 0;
    int totalPenalty = 0;
    int totalCost = 0;
    int totalRisk = 0;

    Collection<BacklogItem> outstandingBacklogItems = this.backlogItemRepository()
        .allOutstandingProductBacklogItems(aTenantId, aProductId);

    // 합산하려면 당연히 For문을 돌아야 겠지만, 이 역시 엄연히 도메인 로직이다.
    for (BacklogItem backlogItem : outstandingBacklogItems) {
      if (backlogItem.hasBusinessPriority()) {
        BusinessPriorityRatings ratings = backlogItem.businessPriority().ratings();

        totalBenefit += ratings.benefit();
        totalPenalty += ratings.penalty();
        totalCost += ratings.cost();
        totalRisk += ratings.risk();
      }
    }

    BusinessPriorityTotals businessPriorityTotals =
        new BusinessPriorityTotals(
            totalBenefit,
            totalPenalty,
            totalBenefit + totalPenalty, // 이 부분은 도메인 로직이고, 애플리케이션 계층으로 유출되어선 안 된다.
            totalCost,
            totalRisk);

    return businessPriorityTotals;
  }
}
```

### 변환 서비스

인프라 내에 위치한 도메인 서비스의 기술적인 구현은 주로 통합(Integration)을 위해 사용된다. 13장에서 자세히 다룰 예정.

### 도메인 서비스의 미니 계층 사용하기

도메인 서비스의 미니 계층을 생성하는 편이 바람직한 경우도 있으나, 종종 애너믹 모델로 이어지기 때문에 안티패턴이다. 

> 트랜잭션과 보안은 애플리케이션 서비스 내에서 애플리케이션의 문제로 다뤄야지, 도메인 서비스 내에서 다루어선 안 된다.

## 서비스의 테스트

서비스 역시 테스트 먼저 짜도 괜찮다.
