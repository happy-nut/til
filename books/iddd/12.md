# 11장. 팩토리

## 도메인 모델 내의 팩토리

애그리게잇 루트가 다른 애그리게잇의 전체 혹은 부분의 인스턴스를 생성하기 위한 팩토리 메소드를 제공한다.
그냥 덜렁 팩토리 클래스가 있는 것보다 이 편이 유비쿼터스 언어를 잘 표현하는 방식이기도 하다.

그러나 생성하는 과정이 너무 복잡하다면(특히 바운디드 컨텍스트를 통합하는 과정에서) 팩토리 메소드를 도메인 서비스에 구현하기도 한다.

만약 도메인 모델 간에 클래스 계층 구조가 나타난다면 추상 팩토리를 이용해볼 수 있다. 계층 구조를 사용함으로써 일어날 고통을 감수해야 할 테지만,
이 경우 서로 다른 타입의 객체를 생성할 수 있는 추상 팩토리의 고전적인 이점을 챙길 순 있다.

## 애그리게잇 루트상의 팩토리 메소드

예를 들어 `Product` 애그리게잇은 `planBacklogItem()` 이라는 메소드를 팩초리로 활용한다. `BacklogItem`은 클라이언트로 다시 반환되는 애그리게잇이다.

### CalendarEntry 인스턴스 생성하기

먼저 테스트를 보자. [여기서](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/test/java/com/saasovation/collaboration/domain/model/calendar/CalendarTest.java#L154-L180) 찾아볼 수 있다. 코드에서
[calendarEntryAggregate](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/test/java/com/saasovation/collaboration/domain/model/calendar/CalendarTest.java#L346-L364) 를 따라가보면 알겠지만, `scheduleCalendarEntry()`는 매개변수가 9개가 필요한데,
[CalendarEntry](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/main/java/com/saasovation/collaboration/domain/model/calendar/CalendarEntry.java#L185-L195) 는 생성자의 매개변수가 11개가 필요하다.
클라이언트가 빠진 2개의 매개변수(`aTenant`, `aCalendarId`)를 전달하지 않는다는 것은 그만큼 생성하는 데 부담이 줄어드는 것을 의미하므로 장점이 된다.

이제 구현을 보자. [여기서](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/main/java/com/saasovation/collaboration/domain/model/calendar/Calendar.java#L99-L123) 찾아볼 수 있다.

`CalendarEntry`는 에그리게잇 루트가 아니므로 생성자가 `protected`로 선언되고, 이는 외부에서 `scheduleCalendarEntry()` 팩토리 메소드를 통해서만
이 `CalendarEntry` 인스턴스를 생성할 수 있도록 강제한다. 따라서 다음 유비쿼터스 언어를 풍부하게 표현할 수 있다.

* `Calendar`가 `CalendarEntry`를 스케줄한다.

모델이 표현력을 획득한다는 것은 장점이지만, 성능 상의 단점이 되기도 한다. 위에서 살펴본 바와 같이 팩토리 메소드를 강제 했기 때문에 `CalendarEntry`를 생성하기 위해선
항상 `Calendar`를 영속성 저장소로부터 가져와야 한다. 이 바운디드 컨텍스트의 트래픽이 너무 증가한다면, 팀은 표현력과 성능 사이에서 저울질을 해봐야 한다.

### Discussion 인스턴스 생성하기

애그리게잇 루트인 `Forum`의 [startDiscussion()](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/main/java/com/saasovation/collaboration/domain/model/forum/Forum.java#L145-L173) 팩토리 메소드를 보자. 
이 구현에서는 `Discussion`을 생성할 뿐만 아니라 `Forum` 이 닫혀 잇는 상황이라면 생성을 막아준다.

이 팩토리 메소드가 표현하고 있는 유비쿼터스 언어는 다음과 같다.

* `Author`가 `Forum`에서 `Discussion`을 시작한다.

클라이언트는 팩토리 메소드 덕분에 [정확하게 표현된 코드](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/test/java/com/saasovation/collaboration/domain/model/forum/DiscussionTest.java#L57-L62) 를 사용할 수 있다.

## 서비스의 팩토리

위에서 도메인 서비스를 팩토리로 활용한다고도 했는데 그 경우를 살펴보자.

[CollaboratorService](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/main/java/com/saasovation/collaboration/domain/model/collaborator/CollaboratorService.java#L19-L30) 는
식별자와 애자일 컨텍스트에서 협업 컨텍스트로의 객체 변환을 제공한다. 협업 객체에서는 "사용자"와 "권한"이라는 개념이 빠지고 "저자", "생성자", "중재자", "소유자", "참석자"
와 같은 개념들이 등장하기 때문에 이에 맞춰 변환할 책임을 이 서비스가 지고 있다.

실제 구현체는 [여기서](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/main/java/com/saasovation/collaboration/port/adapter/service/TranslatingCollaboratorService.java) 확인해볼 수 있다.

바운디드 컨텍스트 마다 쓰이는 유비쿼터스 언어가 다르니, 이를 번역해주는 서비스라는 의미에서 `TranslatingService`라는 이름을 사용한 거구나..! 좋은 표현 방식인 것 같다.
도메인의 관심사는 다른 바운디드 컨텍스트에 대한 내용이나 그 번역이 아니므로 당연히 이 구현체는 인프라 계층에 위치한다.

구현체에서 쓰이는 `UserInRoleAdaptor`의 실체 구현체는 [여기서](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/main/java/com/saasovation/collaboration/port/adapter/service/HttpUserInRoleAdapter.java#L23) 찾아볼 수 있다.
구현 안에서 [CollaboratorTranslator](https://github.com/VaughnVernon/IDDD_Samples/blob/05d95572f2ad6b85357b216d7d617b27359a360d/iddd_collaboration/src/main/java/com/saasovation/collaboration/port/adapter/service/CollaboratorTranslator.java#L22)
를 사용해 Published Language(여기선 JSON) 의 통합 응답을 로컬 모델의 클래스 인스턴스로 변환한다.

클래스 구조가 약간 복잡한데, 책임을 분류하자면 다음과 같다.

* `UserInroleAdaptor`: 외부 바운디드 컨텍스트와의 의사소통을 책임짐
* `CollaboratorTranslator`: 위 결과를 이용해 생성하는 걸 책임짐
