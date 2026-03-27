# Flutter 위젯(Widget)

## 1. 위젯(Widget)이란?

**위젯은 Flutter 앱의 사용자 인터페이스(UI)를 구성하는 기본 단위**입니다.

화면에 보이는 버튼, 텍스트, 이미지 같은 요소뿐 아니라, 레이아웃을 위한 여백, 정렬, 화면 구조, 테마 설정까지도 모두 위젯으로 표현됩니다.

즉, Flutter에서는 **앱 화면을 이루는 거의 모든 것이 위젯**이라고 볼 수 있습니다.

### 예시

- 버튼 → `ElevatedButton`
- 텍스트 → `Text`
- 이미지 → `Image`
- 여백 → `Padding`
- 가운데 정렬 → `Center`
- 화면 기본 구조 → `Scaffold`

---

## 2. Flutter의 UI 방식: 선언형 UI (Declarative UI)

Flutter는 **선언형 UI 방식**을 사용합니다.

이는 “화면을 어떻게 바꿀지”를 일일이 명령하는 방식이 아니라,

**현재 상태(State)에 따라 어떤 UI가 보여야 하는지 선언**하는 방식입니다.

### 동작 방식

1. 현재 상태를 기준으로 UI를 그림
2. 상태가 변경됨
3. Flutter가 변경을 감지함
4. 필요한 위젯을 다시 빌드(Rebuild)함
5. 새로운 상태에 맞는 화면으로 갱신됨

### 예시

카운터 앱에서 숫자가 0에서 1로 바뀌면,

Flutter는 “숫자 텍스트를 1로 보여주는 UI”를 다시 빌드해서 화면에 반영합니다.

---

## 3. 위젯의 핵심 종류: 상태 유무에 따른 분류

Flutter 위젯은 크게 두 가지로 나뉩니다.

- `StatelessWidget`
- `StatefulWidget`

---

### 3-1. StatelessWidget (상태가 없는 위젯)

**StatelessWidget은 내부 상태가 변하지 않는 위젯**입니다.

한 번 화면에 그려지면, 위젯 자체가 스스로 값을 바꾸며 화면을 갱신하지 않습니다.

### 특징

- 화면 표시 내용이 고정적임
- 외부에서 전달받는 값에 따라 표시될 수는 있음
- 자체적으로 상태를 바꿔 다시 그리지는 않음

### 사용 예

- 제목 텍스트
- 고정된 아이콘
- 변하지 않는 안내 문구
- 단순한 화면 구성 요소

### 대표 위젯

- `Text`
- `Icon`
- `Container`

### 예시 코드

```
class MyTitle extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Text('안녕하세요');
  }
}
```

---

### 3-2. StatefulWidget (상태가 있는 위젯)

**StatefulWidget은 실행 중 상태가 바뀔 수 있는 위젯**입니다.

사용자의 입력, 터치, 데이터 변경 등에 따라 화면이 달라져야 할 때 사용합니다.

### 특징

- 내부 데이터가 변경될 수 있음
- 상태가 바뀌면 `setState()`를 호출해 UI를 다시 그림
- 동적인 화면 구현에 적합함

### 사용 예

- 체크박스 선택 여부
- 텍스트 입력창 내용
- 슬라이더 값 변화
- 탭 선택 상태
- 로딩 여부에 따른 화면 전환

### 대표 위젯

- `Checkbox`
- `TextField`
- `Slider`
- `Form`

### 예시 코드

```
class CounterWidget extends StatefulWidget {
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int count = 0;

  void increase() {
    setState(() {
      count++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('$count'),
        ElevatedButton(
          onPressed: increase,
          child: Text('증가'),
        ),
      ],
    );
  }
}
```

---

## 4. StatelessWidget과 StatefulWidget의 차이

| 구분 | StatelessWidget | StatefulWidget |
| --- | --- | --- |
| 상태 변경 | 불가능 | 가능 |
| UI 갱신 | 외부 값이 바뀌어 다시 빌드될 때 | `setState()` 등으로 직접 갱신 |
| 용도 | 정적인 화면 | 동적인 화면 |
| 예시 | 제목, 아이콘, 설명 텍스트 | 입력창, 체크박스, 토글, 카운터 |

---

## 5. 위젯 트리(Widget Tree)란?

Flutter에서는 위젯을 서로 중첩해서 화면을 구성합니다.

이렇게 **부모-자식 관계로 연결된 위젯 구조**를 **위젯 트리(Widget Tree)** 라고 합니다.

### 예시 구조

```
Center(
  child: Column(
    children: [
      Text('Hello Flutter'),
      ElevatedButton(
        onPressed: () {},
        child: Text('클릭'),
      ),
    ],
  ),
)
```

이 코드는 다음과 같은 트리 구조를 가집니다.

- `Center`
    - `Column`
        - `Text`
        - `ElevatedButton`

### 왜 중요한가?

- Flutter UI는 위젯 트리로 관리됨
- 부모 위젯이 자식 위젯의 배치와 스타일에 영향을 줄 수 있음
- 앱 구조를 이해하고 디버깅할 때 매우 중요함

---

## 6. 합성(Composition): Flutter UI의 핵심 설계 방식

Flutter는 복잡한 UI를 만들 때, 거대한 위젯 하나를 만드는 대신

**작고 단순한 위젯들을 조합해서 화면을 구성**합니다.

이 방식을 **합성(Composition)** 이라고 합니다.

### 예시

- `Padding`으로 여백 추가
- `Center`로 가운데 정렬
- `Column`으로 세로 배치
- `Container`로 배경과 테두리 설정

이처럼 필요한 기능을 가진 위젯들을 조립해 원하는 UI를 만듭니다.

### 장점

- 재사용성이 높음
- 유지보수가 쉬움
- 구조를 이해하기 쉬움
- UI를 작은 단위로 나누어 개발 가능

---

## 7. 공식 문서 기준 주요 위젯 카테고리

Flutter는 매우 많은 위젯을 제공하며, 보통 아래와 같이 분류할 수 있습니다.

---

### 7-1. 기본 위젯 (Basic Widgets)

가장 자주 사용하는 핵심 위젯들입니다.

### `Text`

문자열을 화면에 출력하는 위젯입니다.

```
Text('안녕하세요')
```

### `Container`

여백, 크기, 배경색, 테두리 등을 설정할 수 있는 다목적 박스 위젯입니다.

```
Container(
  padding: EdgeInsets.all(16),
  color: Colors.blue,
  child: Text('박스 안 텍스트'),
)
```

### `Row`

자식 위젯들을 **가로 방향**으로 배치합니다.

### `Column`

자식 위젯들을 **세로 방향**으로 배치합니다.

### `Image`

이미지를 보여주는 위젯입니다.

---

### 7-2. 레이아웃 위젯 (Layout Widgets)

위젯의 배치와 크기를 조정하는 데 사용됩니다.

### `Padding`

자식 위젯 주변에 여백을 추가합니다.

### `Center`

자식 위젯을 중앙에 배치합니다.

### `Expanded`

`Row`나 `Column` 안에서 남은 공간을 채우도록 확장합니다.

### `SizedBox`

고정된 크기 또는 간격을 줄 때 사용합니다.

### `Stack`

위젯을 겹쳐서 배치합니다.

예: 이미지 위에 텍스트 올리기

### `Align`

자식 위젯의 위치를 정렬합니다.

---

### 7-3. 입력 및 상호작용 위젯

사용자의 터치, 입력, 선택 등을 처리합니다.

### 대표 위젯

- `ElevatedButton`
- `TextButton`
- `IconButton`
- `TextField`
- `Checkbox`
- `Radio`
- `Switch`
- `Slider`

이 위젯들은 대부분 사용자 액션에 따라 상태가 바뀌므로 `StatefulWidget`과 함께 자주 사용됩니다.

---

### 7-4. 구조/화면 구성 위젯

앱 화면의 기본 골격을 잡는 데 사용됩니다.

### `Scaffold`

Material Design 스타일 앱 화면의 기본 틀을 제공합니다.

포함 가능한 요소:

- `appBar`
- `body`
- `floatingActionButton`
- `drawer`
- `bottomNavigationBar`

### `AppBar`

상단 앱 바를 구성합니다.

### `SafeArea`

노치, 상태바, 홈 인디케이터 등을 피해 안전한 영역에 UI를 배치합니다.

---

### 7-5. 디자인 시스템 위젯

Flutter는 플랫폼 스타일에 맞는 UI를 제공할 수 있습니다.

### Material Components

구글의 **Material Design**을 따르는 위젯입니다.

주로 안드로이드 스타일 UI에 사용됩니다.

대표 예:

- `Scaffold`
- `AppBar`
- `FloatingActionButton`
- `Card`
- `ListTile`

### Cupertino

애플의 **iOS 디자인** 스타일을 따르는 위젯입니다.

대표 예:

- `CupertinoNavigationBar`
- `CupertinoButton`
- `CupertinoPageScaffold`

---

## 8. 알아두면 좋은 개념들

### 8-1. Build 메서드

모든 위젯은 `build()` 메서드를 통해 화면에 어떤 UI를 그릴지 정의합니다.

```
@override
Widget build(BuildContext context) {
  return Text('Hello');
}
```

즉, `build()`는

**“이 위젯은 현재 상태에서 어떤 모양이어야 하는가?”** 를 반환하는 함수입니다.

---

### 8-2. BuildContext

`BuildContext`는 위젯이 위젯 트리에서 어디에 위치하는지에 대한 정보를 제공합니다.

주로 다음과 같은 작업에 사용됩니다.

- 테마 가져오기
- 화면 이동(Navigation)
- 부모 위젯 정보 참조
- `MediaQuery`로 화면 크기 확인

---

### 8-3. Key

위젯이 많아지고 리스트나 동적 변경이 많아질 때, Flutter가 위젯을 정확히 식별하도록 돕는 값입니다.

특히 리스트 재정렬, 삭제, 삽입 시 중요합니다.

---

## 9. 위젯을 잘 사용하는 실전 팁

### 1) 가능한 한 작은 위젯으로 나누기

UI를 한 파일에 길게 작성하기보다, 의미 있는 단위로 분리하면 관리가 쉬워집니다.

### 2) 상태가 필요 없으면 StatelessWidget 사용

불필요하게 StatefulWidget을 쓰면 코드가 복잡해질 수 있습니다.

### 3) 레이아웃 위젯을 익숙하게 사용하기

Flutter UI 작성의 핵심은 `Row`, `Column`, `Padding`, `Expanded`, `Stack` 같은 레이아웃 위젯에 익숙해지는 것입니다.

### 4) UI는 조립한다는 관점으로 접근하기

Flutter에서는 “특수한 위젯을 찾는 것”보다

“기본 위젯을 어떻게 조합할지”를 생각하는 것이 더 중요합니다.

### 5) rebuild를 두려워하지 않기

Flutter는 필요한 부분만 효율적으로 다시 그리도록 최적화되어 있으므로, 기본적인 rebuild는 자연스러운 동작입니다.

---

## 10. 정리

Flutter에서 **위젯은 UI의 전부**라고 할 수 있습니다.

버튼, 텍스트, 배치, 여백, 화면 구조까지 모두 위젯으로 표현됩니다.

핵심은 다음과 같습니다.

- Flutter UI는 **위젯으로 구성된다**
- 위젯은 **StatelessWidget**과 **StatefulWidget**으로 나뉜다
- 화면은 **위젯 트리** 형태로 구성된다
- Flutter는 작은 위젯들을 조합하는 **합성(Composition)** 방식으로 UI를 만든다
- `Row`, `Column`, `Container`, `Padding`, `Scaffold` 등은 매우 자주 쓰이는 핵심 위젯이다