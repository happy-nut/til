# 9장. 모듈

모듈은 자바의 package를 떠올리면 된다.

## 모듈로 설계하기

모듈은 서로 간에 높은 응집도를 갖고 있는 도메인 객체를 담는 이름이 붙여진 컨테이너 역할을 해야 한다.

* 모델링 개념에 맞춰 모듈을 설계하자

    일반적으로 애그리게잇 당 하나의 모듈을 갖게 된다.

* 유비쿼터스 언어에 맞춰 모듈을 명명하자

    DDD의 기본 목적이자 자연스레 다다르는 결과이기도 하다.

* 모델에서 사용하는 일반적인 컴포넌트 타입이나 패턴에 따라서 기계적으로 모듈을 생성하지 말자

    모든 애그리게잇을 하나의 모듈로, 모든 도메인 서비스를 다른 하나의 모듈로, 모든 팩토리를 또 다른 모듈로 분리하면 모듈화의 장점을 전혀 취할 수 없다.
    모듈 조차 풍부한 개념을 말할 수 있어야 한다.

* 느슨하게 결합된 모듈을 설계하자

    모듈 사이 독립성을 최대한 보장해 느슨하게 결합된 클래스와 같은 이점을 취할 수 있다.
  
* 결합이 필요하다면 두 모듈 사이에 circular dependency 가 생기지 않도록 하자

    의존성이 한 방향으로 흐르게 해야 컴포넌트 간 커플링을 줄일 수 있다.

* 자식과 부모 모듈사이 규칙은 느슨하게 하자

    가능하다면 circular dependency를 만들지 않되, 피하기 힘들다면 서로 직접 의존하지 않도록 하자. (e.g. 부모는 자식을 생성하고, 자식은 부모를 식별자를 통해 접근)
  
* 모듈을 모델의 정적인 개념에 따라 만들지 말고 모듈이 담고 있는 객체에 맞추자

    모델의 개념이 변하면 그 모델을 담고 있는 모듈도 이름이 변경되어야 한다.

## 기본 모듈 명명 규칙

자바에서 모듈의 이름은 이름 충돌을 피하기 위해 계층적 형태를 반영한다.

```java
com.yourcompany
```

## 모델을 위한 모듈 명명 규칙

기본 모듈의 다음에 이어질 이름은 바운디드 컨텍스트의 이름으로 하자.

```java
com.yourcompany.agileprojectmanagement
```

위 이름은 정확하긴 하지만, 불필요한 잡음처럼 너무 길다.

```java
com.yourcompany.agilepm
```

제품 이름이나 브랜드를 모듈 이름에 사용하지 않는다는 사실에 주목하자. 제품 이름은 보통 바운디드 컨텍스트와 연관이 없는 경우가 많고, 심지어 종종 바뀐다.

그 다음은 헥사고날 아키텍처의 계층을 나타낸다.

```java
com.yourcompany.agilepm.domain

com.yourcompany.agilepm.domain.model // 여기에 도메인 모델들을 놓는다.
```

## 애자일 프로젝트 관리 컨텍스트의 모듈

모듈 이름이 유비쿼터스 언어를 표현한다.

```java
com.yourcompany.agilepm.domain.model.product
// Product (Aggregate Root)가 있다. 유비쿼터스 언어로는 '제품'이다.

com.yourcompany.agilepm.domain.model.product.backlogItem
// BacklogItem (Aggregate Root)가 있다. 유비쿼터스 언어로는 '제품 백로그 아이템'이다.

com.yourcompany.agilepm.domain.model.product.release
// Release (Aggregate Root)가 있다. 유비쿼터스 언어로는 '제품 릴리즈'다.

com.yourcompany.agilepm.domain.model.product.sprint
// Sprint (Aggregate Root)가 있다. 유비쿼터스 언어로는 '제품 스프린트'다.
```

## 다른 계층 속의 모듈

사용자 인터페이스, 애플리케이션, 도메인, 인프라 외에 다른 자원들은 어떻게 모듈을 나눌까?

```java
com.yourcompany.agilepm.resources
```

## 바운디드 컨텍스트보다 모듈

바운디드 컨텍스트는 너무 두꺼운 결계다. 나누기 애매할 땐, 바운디드 컨텍스트로 나누기 보다 모듈로 먼저 나눠보자.
