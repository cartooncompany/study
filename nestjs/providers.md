# 프로바이더(Provider)란 무엇인가

**프로바이더(Provider)**는 `@Injectable()` 데코레이터가 붙은 클래스로, 서비스·리포지토리·팩토리·헬퍼 등이 여기에 해당합니다. 핵심은 이 클래스가 의존성(dependency)으로 **주입(inject)**될 수 있다는 점이며, 객체들을 서로 연결하는 일은 대부분 Nest 런타임이 맡습니다.

컨트롤러가 HTTP 요청 처리에 집중한다면, 실제 비즈니스 로직은 프로바이더에 위임합니다.

> 참고: https://docs.nestjs.com/providers

---

# 1. 서비스: 대표적인 프로바이더

`@Injectable()`은 이 클래스를 Nest의 **IoC(제어의 역전) 컨테이너**가 관리할 수 있다고 표시하는 역할을 합니다.

```typescript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
```

이제 이 서비스를 컨트롤러 생성자에서 주입받아 씁니다.

```typescript
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

`private` 키워드는 `catsService` 멤버의 **선언과 초기화를 한 줄에** 처리하는 축약 문법입니다.

---

# 2. 의존성 주입(DI)

**의존성 주입(Dependency Injection)**은 Nest의 핵심 디자인 패턴입니다.

Nest는 TypeScript 덕분에 의존성을 **타입 기반**으로 해결합니다. 생성자에 타입만 적으면 Nest가 알아서 인스턴스를 만들어 넣어 줍니다.

```typescript
constructor(private catsService: CatsService) {}
```

이때 기본은 **싱글턴**입니다. 이미 다른 곳에서 요청되어 만들어진 인스턴스가 있으면 그것을 재사용합니다.

---

# 3. 스코프(Scope)

프로바이더는 보통 애플리케이션 수명 주기와 함께합니다. 앱이 부트스트랩될 때 모든 의존성이 해결(=인스턴스화)되고, 앱이 종료될 때 소멸됩니다.

- Node.js는 요청마다 별도 스레드를 쓰는 방식이 아니므로 **싱글턴 사용이 안전**합니다.
- 다만 요청별 캐싱(GraphQL), 요청 추적, 멀티테넌시 같은 경우엔 **요청 스코프(request-scoped)**로 만들어 요청마다 인스턴스를 새로 둘 수도 있습니다.

---

# 4. 주입 방식: 생성자 주입 vs 속성 주입

| 방식 | 코드 | 언제 |
|------|------|------|
| **생성자 주입 (권장)** | `constructor(private svc: Svc) {}` | 대부분의 경우 |
| **속성 주입** | `@Inject('TOKEN') private svc: T;` | 상속 구조에서 `super()`로 전달이 번거로울 때만 |

상속하지 않는 클래스라면 생성자 주입이 어떤 의존성이 필요한지 명확히 드러내므로 더 좋습니다.

```typescript
// 속성 주입 예시
@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

---

# 5. 선택적 프로바이더와 커스텀 프로바이더

## 5.1 선택적 프로바이더

항상 존재하지 않아도 되는 의존성은 `@Optional()`로 표시합니다. 없어도 오류가 나지 않습니다.

```typescript
constructor(@Optional() @Inject('HTTP_OPTIONS') private opts: T) {}
```

## 5.2 커스텀 프로바이더

프로바이더는 클래스뿐 아니라 **값(value)**, **팩토리(동기/비동기)**로도 정의할 수 있습니다. 이때 `'HTTP_OPTIONS'` 같은 **커스텀 토큰**으로 주입합니다.

---

# 6. 프로바이더 등록

프로바이더는 모듈의 `providers` 배열에 등록해야 주입할 수 있습니다.

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

---

# 7. 한눈에 정리

- 프로바이더는 `@Injectable()`이 붙은 클래스로, IoC 컨테이너가 관리하고 다른 곳에 **주입**된다.
- 컨트롤러는 HTTP 처리, 프로바이더는 **비즈니스 로직**을 담당한다.
- 의존성은 **타입 기반**으로 해결되며 기본은 **싱글턴**이다.
- 주입은 주로 **생성자 주입**을 쓰고, 상속 구조에서만 속성 주입을 고려한다.
- 프로바이더는 모듈의 **`providers`에 등록**해야 동작한다.
