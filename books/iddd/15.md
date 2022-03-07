# 14장. 애플리케이션

애플리케이션은 핵심 도메인 모델과 상호 교류하며 이를 지원하기 위해 잘 조합된 컴포넌트의 집합이다.

## 사용자 인터페이스

어떻게 도메인 객체를 유리 같은 사용자 인터페이스 위에 렌더링할 것인가? 그리고 어떻게 사용자 제스처를 모델로 다시 의사통할 것인가?

### 도메인 객체의 렌더링

사용자가 조회할 떄는 여러 애그리게잇 인스턴스로부터 뷰를 만들지만, 사용자가 수정을 요청할 때는 하나의 애그리게잇 인스턴스만을 수정하도록 요청한다.

### 애그리게잇 인스턴스로부터 데이터 전송 객체를 렌더링하기

DTO Assembler가 애그리게잇의 모든 파트에 접근해서 DTO를 만든다. 이 떄 DTO는 뷰에 표시되어야 하는 모든 특성을 갖도록 디자인된다.

단점은 여러가지다.

* YAGNI 에 위배되어 돌발적인 복잡성(도메인 객체는 아니면서 비슷하게 생긴 놈들이 생김)으로 이어지게 될 수 있다.
* 잠재적으로 크기가 커질 수 있는 객체의 추가적인 인스턴스화가 필요하다.
* 애그리게잇이 DTO Assembler 가 필요한 데이터를 쿼리할 수 있으면서 내부 구조를 드러내지 않도록 신중히 설계되어야 한다.

### 애그리게잇 내부 상태를 발행하기 위해 중재자를 사용하자

중재자(Mediator) 인터페이스를 설계해 애그리게잇이 내부 상태를 발행하도록 한다. 이 패턴은 더블 디스패치-콜백 패턴이라고도 불린다.

```java
public class BacklogItem ... {
  ...
  public void provideBacklogItemInterest(BacklogItemInterest anInterest) {
    anInterest.informTenantId(this.tenantId().id());
    anInterest.informProductId(this.productId().id());
    anInterest.informBacklogItemId(this.backlogItemId().id());
    anInterest.informStory(this.story());
    anInterest.informSummary(this.summary());
    anInterest.informType(this.type().toString());
    ...
  }
  
  public void provideTasksInterest(TasksInterest anInterest) {
    Set<Task> tasks = this.allTasks();
    anInterest.informTaskCount(tasks.size());
    for (Task task: tasks) {
      ...
    }
  }
  ...
}
```

위 코드 외에 다른 코드는 책에 안나오긴 하지만 상상으로 구현을 더 해보자면,

```java
interface BacklogItemInterest {
  public void informTenantId(String tenantId);
  public void informProductId(String productId);
  ...
}

interface TasksInterest {
  public void informTaskCount(long taskCount);
  ...
}
```

```java
public class SomeViewMediator implements BacklogItemInterest, TasksInterest {
  String tenantId;
  String productId;
  String backlogItemId;
  int story;
  String summary;
  String type;
  long taskCount;
  
  @Override
  public void informTenantId(String tenantId) {
    this.tenantId = tenantId;
  }
  
  @Override
  public void informProductId(String productID) {
    this.productId = productId;
  }
  ...
  @Override
  public void informTaskCount(long taskCount) {
    this.taskCount = taskCount;
  }
}
```

사용처는 다음처럼 될 것 같다. (근데 이러면 왜 더블 디스패치인지 모르겠네. 그냥 싱글 아닌가...)

```java
SomeViewMediator mediator = new SomeViewMediator();
backlogItem.provideBacklogItemInterest(mediator);
backlogItem.provideTasksInterest(mediator);
```

단점은 다음과 같다.

* 이 접근이 애그리게잇의 책임 안에 있어야 하는가? 하는 의구심이 남는다(자연스러운 도메인의 확장이라고 생각하는 사람들도 있다).
어쨌든 의구심이 남는다면 Interest provider 를 빼내어 다른 클래스로 구현할 수도 있다.

### 도메인 페이로드 객체로부터 애그리게잇 인스턴스를 렌더링하라

DTP(Domain Payload Object) 이건 또 뭘까. DPO는 DTO와는 달리 개별 속성들이 아니라 전체 애그리게잇 인스턴스의 참조를 담도록 설계된다.

프레젠테이션 컴포넌트는 DPO의 애그리게잇에게 보여주고자 하는 특성을 문의한다.

단점:

* 사용자 인터페이스가 모델로 스며드는 걸 막기 위해 중재자나 더블 디스패치나 애그리게잇 루트 쿼리 인터페이스 등을 사용해야 한다.
* 지연 로딩에 대한 처리를 따로(즉시 패치한다거나 해서) 해주어야 한다.

### 애그리게잇 인스턴스의 상태 표현

뭔 소린지 모르겠다. 한국말 맞나? 개똥같은 번역 같으니라고.

### 유스케이스 최적 리포지토리 쿼리

애그리게잇 인스턴스를 여러 읽어서 DTO나 DPO로 구성하는 대신, use case optimal query를 사용하는 방법이 있다.
하나 이상의 애그리게잇 인스턴스의 상위 집합인 사용자 지정 객체를 구성해줄 finder query 메소드를 리포지토리에 설계하는 것이다.
쿼리는 유스케이스의 요구사항만을 전문으로 다루는 VO에 동적으로 결과를 배치한다.

근데 이걸 쓰느니 CQRS를 하라고 한다.

### 다수의 개별 클라이언트 처리하기

애플리케이션이 여러 클라이언트(한 놈은 JSON 받고 다른 놈은 CSV 받고 이런 식)를 지원하는 경우라면 데이터 변환기(Data Transformer)를 사용한다.
애플리케이션 서비스가 데이터 변환기를 수용하도록 설계하는 것이다.
각 클라이언트에선 데이터 변환기 타입을 구체적으로 지정해서 자신이 원하는 데이터 포맷을 가져간다.

### 변환 어댑터와 사용자 편집의 처리

사용자가 이제 도메인 데이터를 갖게 되었고, 이를 편집하는 시점이라고 치자.

다음과 같은 presentation model 이 만들어져 있다.

```java
public class BacklogItemPresentationModel extends AbstractPresentationModel {
  private BacklogItem backlogItem;
  private BacklogItemEditTracker editTracker;
  private BacklogItemApplicationService backlogItemAppService;
  
  public BacklogItemPresentationModel(BacklogItem aBacklogItem) {
    super();
    this.backlogItem = backlogItem;
    this.editTracker = new BacklogItemEditTracker(aBacklogItem);
  }
  
  public String getSummary() {
    return this.backlogItem.summary();
  }
  
  public String getStory() {
    return this.backlogItem.story();
  }
  
  ...
  public void changeSummaryWithType() {
    this.backlogItemAppService.changeSummaryWithType(this.editTracker.summary(), this.editTracker.type());
  }
}
```

사용자가 뷰를 컨트롤하면서 발행한 편집 내용을 `editTracker`가 추적하고, 이를 적용하기 위해 애플리케이션 서비스와 소통하는 것은 `BacklogItemPresentationModel`의 책임이다.  

## 애플리케이션 서비스

애플리케이션 서비스는 도메인 모델의 직접적인 클라이언트임과 동시에, 모델의 상태 변환이 원자적으로 영속되도록 하는 책임을 갖는다.

애플리케이션 서비스는 비즈니스 도메인 로직을 담는 도메인 서비스와는 다르다. 얇게 유지하면서 오직 모델로 향하는 태스크의 조율에만 사용해야 한다.

### 애플리케이션 서비스 예제

다음 인터페이스를 갖는 예제 애플리케이션 서비스가 있다.

* 테넌트를 생성한다
* 테넌트를 프로비저닝한다
* 기존의 테넌트를 활성화한다
* 기존의 테넌트를 비활성화한다
* 미래의 사용자에게 등록 초대를 제공한다
* 특정 테넌트를 조회한다

메소드 시그니처에 도메인 일부 타입이 누출될 수 있는데, 이게 싫다면 원시 타입으로 전부 나열하거나, 커맨드 객체를 설계하는 방법이 있다.
모델에서 타입을 제거하면 의존성과 커플링을 피할 수 있지만 VO로부터 거져얻을 수 있는 강력한 타입 검증과 validation을 포기해야 한다.
또, 메소드의 반환타입을 위해 DTO를 제공하려면 돌발적인 복잡성이 수반될 수도 있다.

위처럼 API를 제공하는 것이 아니라 커맨드 핸들러를 두는 방법도 있다. 사실 별 차이는 없지만, 이벤트 소싱이 더해지면 처리량과 확장성을 높여준다.

### 결합이 분리된 서비스 출력

포트와 어댑터 아키텍처에는 애플리케이션의 모든 메소드가 void 로 선언되도록 하고, 포트에서 데이터를 읽어가는 전략이 있다. 이렇게 되면
각 컴포넌트는 오직 읽어들일 입력과 자신의 행동 및 출력을 내보낼 포트만을 이해하면 된다.

포트로 내보내는 일은 애그리게잇의 순수 커맨드 메소드가 반환 값을 만들지 않을 때와 거의 비슷하며, 이 경우 도메인 이벤트가 발행된다.
애그리게잇의 경우, `DomainEventPublisher`가 애그리게잇의 출력 포트다.

```java
...
@Override
@Transactional(readOnly = true)
public void findTenant(TenantId tenantId) {
  Tenant tenant = this.tenantRepository.tenantOfId(tenantId)

  this.tenantIdentityOutputPort().write(tenant);
}
...
```

## 여러 바운디드 컨텍스트 묶기

여러 바운디드 컨텍스트들로부터 도메인 모델들을 얻어와 합치는 상황이라면 애플리케이션이 유스케이스를 관리하므로 해당 계층에 구현하는 것이 가장 편리하다.
그러나 선을 잘 그어야 한다.

* 새로운 통일된 모델의 바운디드 컨텍스트를 생성할 것인지
* 다수의 바운디드 컨텍스트를 하나의 사용자 인터페이스로 구성할 것인지

## 인프라

인프라는 기술적인 기능을 제공한다. 애플리케이션 서비스는 도메인 모델의 리포지토리 인터페이스에 의존적이지만, 인프라의 구현을 사용한다.

## 엔터프라이즈 컴포넌트 컨테이너

애플리케이션 서비스의 사용을 용이하게 하기 위해 EJB(엔터프라이즈 자바빈)이나 스프링 같은 DI 컨테이너가 제공하는 기능을 활용할 수 있다.
