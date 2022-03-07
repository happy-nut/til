# 6장. 값 객체

가능하다면 최대한 값 객체를 써야 한다. Entity보다 생성, 테스트, 사용, 초적화, 유지 관리가 더 쉽기 때문이다.

## 값의 특징

도메인 개념을 값 객체로 모델링 할 때는 유비쿼터스 언어를 확실히 활용해야 한다. 이게 가장 중요하다.
이 대원칙 아래 Value Object의 특징을 하나씩 살펴보자.

### 측정, 수량화, 설명

값 객체는 도메인 안에 있는 존재가 아니라 도메인 자체를 측정하고 수량화하고 설명하는 개념이다. 이를테면 사람의 나이.

### 불변성

Value Object는 생성되면 변경할 수 없다. 기호에 따라 Value Object가 entity에 대한 참조를 가지고 있을 수 있으나, 엔티티가 상태를 변경할 때 불변성을 위반하므로
예외적인 경우로 생각해야 한다. 단지 해당 경우는 불변성, 표현성, 편리함을 위해서만 쓰인다는 사고방식을 지녀야 한다.

### 개념적 전체

```java
public class Thing {
  private String name;
  private BigDecimal amount;
  private String currency;
}
```

위 코드에서 amount와 currency 는 개념적으로 묶인다. 따라서 다음과 같이 설계하는 것이 더 적절하다.

```java
public class Thing {
  private String name;
  private MonetaryValue worth;
}
```

단 `worth`라는 저 속성 이름은 반드시 유비쿼터스 언어를 만든 후에야만 정할 수 있다.

만약 name의 첫번째 글자가 항상 대문자여야 한다면, `ThingName`과 같이 새로운 VO를 정의하는 것이 좋다.
이 때 만약 루비나 코틀린 처럼 mixin으로 클래스를 패치해서 쓰는 경우는 어떨까?

```kotlin
fun String.toThingName() {}
```

이건 별로 좋은 방법이 아니다. `toThingName()` 을 모든 `String` 에게 열어주는 것도 이상하며, 무엇보다 유비쿼터스 언어와 거리가 멀다. 만약 파라미터를 받아야 하는 상황이라면
더더욱 그렇다.

### 대체성

값은 대체할 수 있다. 현재 상태가 올바르지 않은 상황이 왔다면, 새로운 값으로 전체를 완전히 대체해야 한다.

```java
int total = 3; // 단순 숫자뿐 아니라 VO도 마찬가지로 취급한다.
total = 4;
```

### 값 등가성

VO의 등가성은 두 객체의 타입과 특성을 비교해서 결정된다.

### 부작용이 없는 행동

VO의 메소드는 부작용이 없는(상태를 변경하지 않고 출력을 만들어 내는) 함수로 설계할 수 있다.

```java
public FullName withMiddleInitial(String aMiddleNameOrInitial) {
  if (aMiddleNameOrInitial == null) {
    throw new IllegalArgumentException("Must provide a middle name or initial");
  }
  
  String middle = aMiddleNameOrInitial.trim();
  if (middle.isEmpty()) {
    throw new IllegalArgumentException("Must provide a middle name or initial");
  }
  
  return new FullName(this.firstName(), middle.substring(0, 1).toUpperCase(), this.lastName());
}
```

사용 시:

```java
FullName name = new FullName("Happy", ""nut);

name = name.withMiddleInitial("S");
```

만약 값 객체가 엔티티에 대해 의존하고 있는 경우는 어떨까? 다음은 VO인 `businessPriority`가 파라미터로 Entity인 `product`를 받고 있는 상황이다.

```java
float priority = businessPriority.priorityOf(product);
```

다음과 같은 구조가 더 낫다. 유비쿼터스 언어에서 비즈니스 중요도가 표현될 때 `float` 말고 커스텀 타입을 정의해볼 수도 있겠다.

```java
float priority = businessPriority.priority(product.businessPriorityTotals());
```

## 미니멀리즘으로 통합하기

모든 DDD프로젝트에는 다수의 바운디드 컨텍스트가 있으며, 컨텍스트를 통합하는 올바른 방법을 찾아야 한다.

식별자와 액세스 컨텍스트에서 `User`와 `Role`은 애그리게잇이지만, 협업 컨텍스트에서 이에 해당하는 `Moderator`는 값 객체다.
통합을 할 때는 이런 식으로 ACL에서 값 객체로 변환해 주는 것이 좋다.

물론 바운디드 컨텍스트를 넘어서까지 변경을 유지 관리해야 한다면 엔티티를 사용해야 할 수도 있겠지만, 가능하면 그런 설계는 피하는 것이 좋다.

## 값으로 표현되는 표준 타입

`PhoneNumber`가 있다고 하자. 집전화인지 휴대폰 번호인지, 회사 번호인지 모른다. 이 때 각 상황마다 별도의 VO 클래스를 만드는 편이 좋을까?
아니다. `Home`, `Mobile`, `Work`, `Other` 등으로 전화의 타입을 나타내는 편이 좋다. 전화의 표준 타입을 나타내기 때문이다.

```java
public enum GroupMemberType {
  GROUP {
    public boolean isGroup() { return true; }
  },
  
  USER {
    public boolean isUser() { return true; }
  };
  
  public boolean isGroup() { return false; }
  public boolean isUser() { return false; }
}
```

다음은 위 표준 타입을 활용하는 `User`의 메소드다.

```java
protected GroupMember toGroupMember() {
  GroupMember groupMemeber = new GroupMember(this.tenantId(),  this.username(), GroupMemberType.USER);
  
  return groupMember
}
```

## 값 객체의 테스트

VO는 테스트 주도 개발과 궁합이 좋다. 객체를 사용하는 방법에 대해 예시를 제공하면서 도메인 모델의 설계를 이끌어내기 때문이다.
단 도메인 모델의 테스트는 도메인 전문가의 입장에서 의미가 있어야 한다.

VO의 불변성을 테스트 하기 위해 객체를 복사해놓고, 객체의 메소드를 테스트 후 마지막에 복사된 객체와 오리지널 객체가 같은지 비교해보는 방법이 있다.

## 구현

저자는 보통 값 객체마다 적어도 2개의 생성자를 지원한다. 첫 번째 생성자는 생성용으로 매개변수 전체를 받고, 두번째 생성자는 얕은 복사용으로 생성한다.

부작용 없는 함수의 메소드 이름은 의도적으로 `get` 접두어를 제거한다. `getValuePercentage()`는 기술적인 컴퓨터 명령이지만 `valuePercentage()`는
사람이 충분히 유창하게 읽을 수 있기 때문이다.

`equals()` 를 구현할 때는 타입과 프로퍼티를 검사한다. java의 경우 `hashCode()`도 같이 구현해주어야 한다.

## 값 객체의 저장

### 데이터 모델 누수의 부정적 영향을 거부하라

값 객체를 데이터 저장소로 저장하는 대부분의 경우 비정규화된 방식(데이터베이스 테이블 행)으로 저장된다. 이 때,
데이터 모델은 부타적이어야 한다. 도메인 모델을 위해 데이터 모델을 선계해야 하지, 주객전도가 되어서는 안된다는 말이다.

VO가 영속성 저장소의 엔티티로 저장되어야 할 때가 있다. ORM을 통해 값 객체 인스턴스의 컬렉션을 지원하는 경우가 그러하다.
그러나 위에 말한 바와 같이 영속성 메커니즘에서 Entity가 되어야 한다고 VO을 Entity로 바꾸어선 안된다. 

### ORM과 단일 값 객체

VO가 Entity의 프로퍼티라면, Entity가 영속화되어 매핑되는 테이블에 VO의 각 프로퍼티들도 컬럼에 매핑되어 저장된다. 임피던스 불일치가 발생하긴 하지만,
기능적이고 최적화되었다.

### ORM과 한 열로 직렬화되는 여러 값

Entity가 VO의 List나 Set을 프로퍼티로 가지고 있는 경우라면 텍스트로 변환하여 직력화한 후 한 컬럼으로 저장을 해볼 수 있다.

그러나 이 경우 몇 가지 잠재적 단점을 감안해야 한다.

* 직력화된 값의 크기가 컬럼의 최대 크기를 넘어설 수도 있는 문제
* 값 객체를 통해 쿼리해야 하는 경우(그러나 이 경우는 드물다)

### ORM과 데이터베이스 엔티티로 지원되는 여러 값

VO를 데이터 모델의 엔티티로 취급하는 방법인데, 이런 접근법은 임피던스 불일치에 따라 요구될 뿐이지, DDD원칙에 따라 요구되는 사항이 아니다.
어쨌든 이걸 해야만 하는 경우라면 이를 실천하기 위해 대리키 접근법을 사용해볼 수 있다.

```java
public abstract class IdentifiedDomainObject implements Serializable {
  private long id = -1;
  
  public IdentifiedDomainObject() { super(); }
  
  protected long id() {
    return this.id;
  }
  
  protected void setId(long anId) {
    this.id = anId;
  }
}

public abstract class IdentifiedValueObject extends IdentifiedDomainObject {
  public IdentifiedValueObject() {
    super();
  }
}
```

사용 예제는 다음과 같다.

```java
public final class GroupMember extends IdentifiedValueObject {
  private String name;
  private TenantId tenantId;
  private GroupMemberType type;
  
  public GroupMember(TenantId aTenantId, String aName, GroupMemberType aType) {
    this();
    this.setName(aName);
    this.setTenantId(aTenantId);
    this.setType(aType);
    this.initialize();
  }
  ...
}
```

`Group`이라는 애그리게잇 루트 엔티티는 임의의 수의 `GroupMember`를 포함하게 된다.

```java
public class Group extends Entity {
  private String description;
  private Set<GroupMember> groupMembers;
  private String name;
  private TenantId tenantId;
  
  public Group(TenantId aTenantId, String aName, String aDescription) {
    this();
    this.setDescription(aDescription);
    this.setName(aName);
    this.setTenantId(aTenantId);
    this.initialize();
  }
  ...
  protected Group() {
    super();
    this.setGroupMembers(new HashSet<GroupMember>(0));
  }
  
  public void replaceMembers(Set<GroupMember> aReplacementMembers) {
    this.groupMembers().clear();
    this.setGroupMember(aReplacementMembers);
  }
}
```

ORM으로는 `GroupMember`의 `id` 외래키로 join 해서 가져온다.

### ORM과 조인 테이블로 지원되는 여러 값

하이버네이트는 값 타입 자체가 데이터 모델 엔티티 특성을 가질 필요 없이, 다중 값 컬렉션을 조인 테이블에 저장하는 방법을 제공한다. 이건 단순히 컬렉션 값 요소를 해당하는 테이블에 저장하면서
부모 엔티티 도메인 객체의 DB 식별자가 외래 키로 설정된다. (하이버네이트의 `composite-element` 테그)

그러나 이는 제약사항이 너무 많아서 일반적으로는 피해야 할 매핑 접근법이다.

### ORM과 상태로서의 열거형 객체

열거형을 저장하는 문제의 단순한 해답은 해당 텍스트 표현을 저장하는 방법이다.
