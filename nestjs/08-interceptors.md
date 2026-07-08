# 인터셉터(Interceptor)란 무엇인가

**인터셉터(Interceptor)**는 `@Injectable()`이 붙고 `NestInterceptor` 인터페이스를 구현한 클래스입니다. **관점 지향 프로그래밍(AOP, Aspect Oriented Programming)** 기법에서 영감을 받았으며, 핸들러 실행 **전후**를 감싸서 부가 로직을 끼워 넣을 수 있습니다.

인터셉터로 할 수 있는 일:

- 메서드 실행 **전/후**에 추가 로직 바인딩 (로깅, 타이밍 측정 등)
- 함수가 **반환한 결과**를 변환 (응답 감싸기 등)
- 함수가 **던진 예외**를 변환
- 기본 동작을 **확장**
- 특정 조건에서 함수를 **완전히 덮어쓰기** (예: 캐싱)

> 참고: https://docs.nestjs.com/interceptors

---

# 1. intercept() 와 CallHandler

인터셉터의 핵심은 `intercept()` 메서드입니다.

```typescript
intercept(context: ExecutionContext, next: CallHandler): Observable<any>
```

- **`context`**: `ExecutionContext` — 현재 실행 정보. 가드에서 봤던 그것(요청/핸들러/클래스 접근). (→ [[07-guards]])
- **`next`**: `CallHandler` — `handle()` 메서드로 **실제 라우트 핸들러를 호출**하는 열쇠.

## `next.handle()`의 의미

- `handle()`을 **호출해야** 라우트 핸들러가 실행됩니다. **호출하지 않으면 핸들러 자체가 실행되지 않습니다** (→ 캐싱/오버라이드에 활용).
- `handle()`은 **`Observable`을 반환**하므로, RxJS 연산자(`tap`·`map`·`catchError`·`timeout` 등)로 응답 스트림을 자유롭게 가공할 수 있습니다.

---

# 2. 실행 전/후 로직: LoggingInterceptor

`tap`으로 스트림을 건드리지 않고 부수효과(로깅)만 넣습니다.

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

`Before...`는 핸들러 실행 전에, `After... Nms`는 응답이 나온 뒤에 찍힙니다.

---

# 3. 인터셉터 바인딩

`@UseInterceptors()`로 붙입니다. 다른 데코레이터와 마찬가지로 **클래스**(DI·재사용) 또는 **인스턴스**를 넘길 수 있습니다.

| 스코프 | 방법 |
|--------|------|
| 메서드 | 핸들러 위에 `@UseInterceptors(LoggingInterceptor)` |
| 컨트롤러 | 클래스 위에 `@UseInterceptors(LoggingInterceptor)` |
| 전역 | `main.ts`에서 `app.useGlobalInterceptors(new LoggingInterceptor())` |

```typescript
@UseInterceptors(LoggingInterceptor)   // 클래스 → Nest가 인스턴스화 + DI
export class CatsController {}
```

## 전역 인터셉터 + 의존성 주입

전역 인터셉터는 모듈 바깥이라 **DI를 못 씁니다**. 필요하면 `APP_INTERCEPTOR` 토큰으로 모듈에서 등록합니다. (→ [[05-exception-filters]]의 `APP_FILTER`, [[06-pipes]]의 `APP_PIPE`, [[07-guards]]의 `APP_GUARD`와 동일 패턴)

```typescript
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
  ],
})
export class AppModule {}
```

---

# 4. 응답 변환 (map)

## 응답을 `{ data }`로 감싸기 — TransformInterceptor

```typescript
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

핸들러가 `[...]`를 반환하면 클라이언트는 `{ "data": [...] }`를 받습니다. 응답 형식을 앱 전역에서 통일할 때 유용합니다.

## null → 빈 문자열 — ExcludeNullInterceptor

```typescript
@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value));
  }
}
```

---

# 5. 예외 변환 (catchError)

핸들러에서 발생한 예외를 가로채 다른 예외로 바꿉니다.

```typescript
import { throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

> 참고: 예외 자체를 잡아 응답을 완전히 제어하는 건 **예외 필터**의 역할이고, 인터셉터는 스트림 관점에서 예외를 **매핑**한다는 점에서 결이 다릅니다. (→ [[05-exception-filters]])

---

# 6. 스트림 덮어쓰기 (핸들러 실행 생략): CacheInterceptor

`handle()`을 호출하지 않고 **새 스트림**(`of(...)`)을 반환하면 라우트 핸들러가 **아예 실행되지 않습니다**. 캐싱의 핵심 원리입니다.

```typescript
import { of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);      // handle() 미호출 → 핸들러 건너뜀
    }
    return next.handle();
  }
}
```

---

# 7. 타임아웃 처리: TimeoutInterceptor

`timeout` 연산자로 일정 시간 초과 시 요청을 종료하고, `TimeoutError`를 잡아 표준 예외로 바꿉니다.

```typescript
import { throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  }
}
```

5초 안에 응답이 없으면 `408 Request Timeout`으로 응답합니다.

---

# 8. 한눈에 정리

- 인터셉터는 `NestInterceptor`를 구현한 클래스로, **AOP**처럼 핸들러 실행 **전후**를 감싼다.
- `intercept(context, next)`에서 **`next.handle()`** 를 호출해야 핸들러가 실행된다. 호출 안 하면 **핸들러를 건너뛴다**(→ 캐싱/오버라이드).
- `handle()`이 반환하는 **`Observable`** 에 RxJS 연산자를 얹어 결과·예외를 가공한다.
  - `tap` → 전후 부수효과(로깅·타이밍)
  - `map` → 응답 변환(`{ data }` 감싸기, null 처리)
  - `catchError` → 예외 변환
  - `timeout` → 시간 초과 처리
- `@UseInterceptors()`로 메서드/컨트롤러/전역 바인딩. 전역에서 DI가 필요하면 **`APP_INTERCEPTOR`** 토큰으로 모듈에 등록.

## ✅ 셀프 체크

1. 인터셉터가 영감을 받은 프로그래밍 기법은? 할 수 있는 일 3가지 이상 말해보기.
2. `intercept()`에서 `next.handle()`을 호출하지 않으면 어떻게 되나?
3. `handle()`의 반환 타입은? 그래서 무엇을 얹어 결과를 가공하나?
4. 응답을 `{ data: ... }`로 감싸려면 어떤 RxJS 연산자를 쓰나?
5. CacheInterceptor가 캐시 히트 시 핸들러를 건너뛰는 원리는?
