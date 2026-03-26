# 1. BuildContext란 무엇인가

BuildContext는 **위젯 트리에서 현재 위젯의 위치를 나타내는 객체**이다.

즉 BuildContext는 다음을 가능하게 한다.

- 현재 위젯의 **위치 정보**
- **상위 위젯 탐색**
- **InheritedWidget 데이터 접근**
- **Navigator / Theme / MediaQuery 등 환경 정보 접근**

간단히 말하면

> BuildContext = 위젯 트리에서 현재 위치를 기준으로 탐색하기 위한 핸들
> 

---

## 2. BuildContext의 주요 역할

## 1) 상위 위젯 데이터 접근

대표적인 예로는 `Theme.of(context)`가 있다.

```dart
Text(
	"Hello",
	style: Theme.of(context).textTheme.bodyLarge,
)
```

동작 방식

1. 현재 `context` 위치에서 시작
2. 위젯 트리를 위쪽으로 탐색
3. 가장 가까운 Theme 위젯을 찾음
4. 그 Theme 데이터를 반환

---

## 2) 화면 정보 접근(MediaQuery)

```dart
double width = MediaQuery.of(context).size.width;
```

현재 디바이스 정보 접근 가능

예

- 화면 크기
- orientation
- padding
- devicePixelRatio

---

## 3) Navigator 사용

```dart
Navigator.of(context).push(
	MaterialPageRoute(builder: (_) => SecondPage()),
)
```

동작

1. context에서 시작
2. 위쪽으로 탐색
3. 가장 가까운 Navigator를 찾음
4. 그 Navigator를 통해 화면 이동

---

## 4) InheritedWidget 접근

Flutter의 환경 데이터 대부분은 **IngeritedWidget** 기반이다.

대표적인 예:

- Theme
- MediaQuery
- Navigator
- Lacalizations
- Provider

예)

```dart
Provider.of<UserModel>(context);
```

---

# 중요한 특징

## 1) 각 위젯은 자신의 BuildContext를 가진다.

```dart
Widget (설계도)
   ↓
Element (트리 관리)
   ↓
RenderObject (렌더링)
```

즉

```dart
BuildContext = Element 인터페이스
```

그래서 정확히 말하면

> 각 Widget instance는 대응되는 Element(BuildContext)를 가진다.
> 

---

## 2) BuildContext는 위치 기반이다.

예를 들어

```dart
Scaffold.of(context)
```

이 코드는 context가 Scaffold 아래에 있어야 한다.

잘못된 예

```dart
Widget build(BuildContext context) {
	return Scaffold(
		body: ElevatedButton(
			onPressed: () {
				Scaffold.of(context).openDrawer(); //오류 발생 가능
			}
		),
	);
}
```

이유: 현재 context는 Scaffold 위에 있기 때문

해결 방법

```dart
Builder(
	builder: (context) {
		return ElevatedButton(
			onPressed: () {
				Scaffold.of(context).openDrawer();
      },
    );
  },
)
```

Builder가 **새로운 context를 생성**합니다.

# 4. BuildContext 사용 시 주의할 점

## 1) async 이후 context 사용

위젯이 이미 dispose 되었을 수 있습니다.

위험한 코드

```
onPressed: () async {
	await Future.delayed(Duration(seconds:3));
	Navigator.of(context).pop();
}
```

안전한 방법

```
if (!context.mounted)return;
Navigator.of(context).pop();
```

---

## 2) BuildContext를 저장하지 말 것

Flutter 공식 권장

> BuildContext는 오래 저장하지 말 것
> 

예

잘못된 예

```
lateBuildContextsavedContext;
```

이유

- 위젯 트리가 변경되면 context가 invalid 될 수 있음

---

## 3) initState에서 context 사용

중요한 부분입니다.

흔한 오해

> initState에서는 context를 사용할 수 없다
> 

실제 → context는 존재함

하지만 **InheritedWidget 의존 코드는 안전하지 않을 수 있음**

예) 권장되지 않음

```
Theme.of(context)
Provider.of(context)
```

권장되는 위치

```
@override
voiddidChangeDependencies() {
super.didChangeDependencies();
Theme.of(context);
}
```

---

# 5. Flutter 내부 구조와 BuildContext

Flutter 렌더링 구조

```
Widget (설계도)
   ↓
Element (트리 관리)
   ↓
RenderObject (렌더링)
```

여기서

```
BuildContext = Element
```

즉 context는 단순한 객체가 아니라

> **위젯 트리의 실제 노드를 나타내는 객체**
> 

---

# 6. 한 줄 정리

**BuildContext = 현재 위젯의 위치를 기준으로 위젯 트리를 탐색하기 위한 객체**

또는

> **Flutter에서 context는 "현재 위젯의 위치 좌표" 역할을 한다.**
>