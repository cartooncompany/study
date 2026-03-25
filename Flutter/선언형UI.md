# 선언형 UI(Declarative UI)
## 1. 기존 UI 프로그래밍 방식: 명령형(Imperative)

Win32부터 iOS, Android에 이르기까지 많은 프레임워크는 일반적으로 **명령형(Imperative)** 방식의 UI 프로그래밍을 사용한다.

이 방식에서는 `UIView`(또는 이에 상응하는 객체)와 같은 **완전한 기능을 갖춘 UI 객체**를 먼저 생성한다.

그리고 **UI 상태가 변경될 때마다 메서드나 setter를 호출하여 해당 객체를 직접 변경**한다.

즉,

- UI 객체를 생성하고
- 상태가 바뀔 때마다
- 해당 객체의 속성이나 자식 뷰를 직접 수정하는 방식이다.

---

## 2. Flutter가 선언형 UI를 사용하는 이유

Flutter는 개발자가 **다양한 UI 상태 간 전환을 직접 프로그래밍해야 하는 부담을 줄이기 위해** 선언형 방식을 사용한다.

Flutter에서는 다음과 같은 방식으로 동작한다.

- 개발자는 **현재 UI 상태를 설명**한다.
- 실제 UI 변경과 전환은 **프레임워크가 처리**한다.

즉,

`어떻게 바꿀지(How)`가 아니라

`어떤 상태인지(What)`를 선언하는 방식이다.

---

# 선언형 프레임워크에서 UI를 변경하는 방법

다음과 같은 단순화된 UI 구조를 가정해보자.

```
View A
 └─ View B
     ├─ c1
     └─ c2
```

처음에는 `View B`가 **c1, c2 두 개의 하위 뷰**를 가지고 있다.

이후 상태가 변경되어 다음과 같이 바뀐다.

```
View A
 └─ View B
     └─ c3
```

즉, **c1과 c2가 제거되고 c3 하나만 남는 상황**이다.

---

# 1. 명령형 스타일 (Imperative Style)

명령형 방식에서는 보통 `ViewB`의 **소유 객체(owner)**로 이동한 뒤,

`selector`나 `findViewById` 같은 방법으로 **인스턴스 b를 가져온다.**

그 다음 **해당 객체를 직접 수정**한다.

이 과정에서 **암묵적으로 UI가 다시 그려지는(invalidation)** 과정이 발생한다.

예시 코드:

```
 // Imperative style
 b.setColor(red)
 b.clearChildren()
 ViewC c3 = new ViewC(...)
 b.add(c3)
```

또한 UI의 “**진짜 상태(Source of Truth)**”가 `b` 인스턴스보다 더 오래 유지될 수 있기 때문에,

같은 설정을 **ViewB의 생성자에서도 다시 구현해야 하는 상황**이 발생할 수 있다.

---

# 2. 선언형 스타일 (Declarative Style)

선언형 방식에서는 뷰 구성 요소(Flutter의 `Widget` 등)가 다음과 같은 특징을 가진다.

- **불변(immutable)**
- 가벼운 **설계도(blueprint)** 역할

UI를 변경할 때는 기존 객체를 수정하는 대신,

**위젯을 다시 빌드(rebuild)** 한다.

Flutter에서는 일반적으로 `StatefulWidget`에서

`setState()`를 호출하여 이를 수행한다.

그러면 **새로운 위젯 트리(widget tree)가 다시 생성된다.**

예시 코드:

```
 // Declarative style
 return ViewB(color: red, child: const ViewC());
```

여기서는 기존 인스턴스 `b`를 직접 수정하지 않는다.

대신 Flutter가 **새로운 Widget 인스턴스를 생성**하여 UI 상태를 표현한다.

---

# Flutter 내부 동작 방식

전통적인 UI 객체가 담당하던 여러 책임(예: 레이아웃 상태 유지)은

Flutter 내부의 **RenderObject**가 관리한다.

특징은 다음과 같다.

- **RenderObject는 프레임 간에 유지된다**
- **Widget은 가벼운 구조**
- Widget은 **상태 변화에 따라 RenderObject를 어떻게 변경할지 프레임워크에 지시**한다.

따라서,

- 개발자는 **UI 상태를 선언**하고
- **실제 변경 처리와 최적화는 Flutter 프레임워크가 담당**한다.