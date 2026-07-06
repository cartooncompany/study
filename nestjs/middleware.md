# 미들웨어(Middleware)란 무엇인가

**미들웨어(Middleware)**는 라우트 핸들러가 실행되기 **전에** 호출되는 함수입니다. 요청(`req`) 객체와 응답(`res`) 객체, 그리고 요청-응답 주기의 다음 미들웨어 함수인 `next()`에 접근할 수 있습니다.

Nest 미들웨어는 기본적으로 Express 미들웨어와 동일하게 동작합니다.

> 참고: https://docs.nestjs.com/middleware

---

# 1. 미들웨어가 할 수 있는 일

- 임의의 코드를 실행한다.
- `req`/`res` 객체를 수정한다.
- 요청-응답 주기를 종료한다.
- 스택의 다음 미들웨어를 호출한다.
- 응답을 끝내지 않는다면 **반드시 `next()`를 호출**해 제어를 넘겨야 한다. 그렇지 않으면 요청이 멈춘 채로 걸린다.

---

# 2. 두 가지 구현 방식

## 2.1 클래스 미들웨어

`@Injectable()`을 붙이고 `NestMiddleware` 인터페이스를 구현하며 `use()` 메서드를 둡니다. 프로바이더처럼 **의존성 주입(DI)**을 지원합니다.

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

## 2.2 함수형 미들웨어

멤버·의존성이 없다면 그냥 함수로 만드는 편이 간단합니다.

```typescript
export function logger(req: Request, res: Response, next: NextFunction) {
  console.log('Request...');
  next();
}
```

의존성이 필요 없을 때는 언제든 함수형 미들웨어를 우선 고려합니다.

---

# 3. 적용하기: `configure()` 메서드

`@Module()` 데코레이터에는 미들웨어를 넣을 자리가 없습니다. 대신 모듈이 `NestModule` 인터페이스를 구현하고 `configure()` 메서드에서 설정합니다.

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';

@Module({ imports: [CatsModule] })
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');   // /cats 경로에 적용
  }
}
```

`configure()`는 `async/await`로 비동기로 만들 수도 있습니다.

---

# 4. MiddlewareConsumer

`MiddlewareConsumer`는 미들웨어를 관리하는 헬퍼 클래스로, 메서드를 체이닝(fluent style)해서 씁니다.

| 메서드 | 역할 |
|--------|------|
| `.apply(A, B, ...)` | 미들웨어 지정 (여러 개면 순차 실행) |
| `.forRoutes(...)` | 적용 대상: 경로 문자열 / `RouteInfo` 객체 / **컨트롤러 클래스** |
| `.exclude(...)` | 특정 라우트 제외 |

```typescript
// 특정 메서드로 제한
.forRoutes({ path: 'cats', method: RequestMethod.GET });

// 컨트롤러로 지정 (가장 많이 씀)
.forRoutes(CatsController);

// 여러 미들웨어 순차 실행
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);

// 특정 라우트 제외
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    'cats/{*splat}',
  )
  .forRoutes(CatsController);
```

---

# 5. 라우트 와일드카드

- 이름 있는 와일드카드 `'abcd/*splat'` → `abcd/1`, `abcd/abc` 등 매칭 (`splat`은 그냥 이름일 뿐).
- `abcd/` 자체(뒤 문자 없음)까지 매칭하려면 중괄호로 감싸 선택적으로: `'abcd/{*splat}'`.

---

# 6. 전역 미들웨어

등록된 모든 라우트에 한 번에 적용하려면 `main.ts`에서 `app.use()`를 씁니다.

```typescript
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```

주의: 전역 미들웨어(`app.use()`)에서는 **DI 컨테이너에 접근할 수 없습니다**. 이 경우 함수형 미들웨어를 쓰거나, 클래스 미들웨어를 모듈에서 `.forRoutes('*')`로 적용합니다.

---

# 7. 한눈에 정리

- 미들웨어는 **라우트 핸들러 실행 전에** 호출되며 `req`/`res`/`next`에 접근한다.
- 응답을 끝내지 않으면 반드시 **`next()`를 호출**한다.
- **클래스(DI 지원)**와 **함수형(간단)** 두 방식이 있으며, 의존성이 없으면 함수형을 우선한다.
- 등록은 `providers`가 아니라 **모듈의 `configure()`** 에서 `MiddlewareConsumer`로 한다.
- 전역 적용은 `app.use()`지만, 여기선 DI를 못 쓰는 점에 주의한다.
