# 예외 필터(Exception Filter)란 무엇인가

Nest에는 애플리케이션 전반에서 처리되지 않은 모든 예외를 담당하는 내장 **예외 계층(exceptions layer)**이 있습니다. 코드에서 예외가 처리되지 않으면 이 계층이 잡아서 사용자 친화적인 응답을 자동으로 보냅니다.

기본 동작은 내장 **전역 예외 필터**가 수행하며, `HttpException`(및 그 하위 클래스)을 처리합니다. 인식되지 않는 예외는 아래 기본 응답으로 변환됩니다.

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> 참고: https://docs.nestjs.com/exception-filters

---

# 1. 표준 예외 던지기

`@nestjs/common`의 `HttpException`을 던지면 표준 HTTP 응답으로 변환됩니다.

```typescript
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
// →  { "statusCode": 403, "message": "Forbidden" }
```

`HttpException` 생성자의 인자:

- **`response`** (필수): JSON 응답 본문. 문자열이면 `message`만, 객체면 본문 전체를 재정의.
- **`status`** (필수): HTTP 상태 코드. `HttpStatus` 열거형 사용 권장.
- **`options`** (선택): `cause`로 내부 원인 오류를 전달. 응답엔 직렬화되지 않고 **로깅용**으로 유용.

```typescript
throw new HttpException(
  { status: HttpStatus.FORBIDDEN, error: 'This is a custom message' },
  HttpStatus.FORBIDDEN,
  { cause: error },
);
```

---

# 2. 내장 HTTP 예외

`HttpException`을 상속한 표준 예외들이 준비되어 있어 대부분 직접 만들 필요가 없습니다.

`BadRequestException` · `UnauthorizedException` · `NotFoundException` · `ForbiddenException` · `ConflictException` · `UnprocessableEntityException` · `InternalServerErrorException` · `ImATeapotException` 등.

```typescript
throw new NotFoundException('게시글을 찾을 수 없습니다');

// cause + description 함께 제공
throw new BadRequestException('Something bad happened', {
  cause: new Error(),
  description: 'Some error description',
});
// →  { "message": "Something bad happened", "error": "Some error description", "statusCode": 400 }
```

---

# 3. 커스텀 예외

만들어야 한다면 `HttpException`을 상속해 예외 계층 구조를 두는 게 좋습니다. 그래야 Nest가 인식하고 자동 처리합니다.

```typescript
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

> 참고: 내장 예외(`HttpException` 등)는 정상 흐름의 일부로 취급되어 **콘솔에 로깅되지 않습니다**. 로깅하려면 커스텀 예외 필터가 필요합니다.

---

# 4. 예외 필터로 완전히 제어하기

응답 형식을 통일하거나 로깅을 넣는 등 예외 계층을 **완전히 제어**하고 싶을 때 예외 필터를 만듭니다.

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

- `ExceptionFilter<T>` 인터페이스를 구현하고 `catch(exception, host)` 메서드를 제공.
- **`@Catch(HttpException)`**: 이 필터가 잡을 예외 타입 지정. 쉼표로 여러 타입 지정 가능.
- **`ArgumentsHost`**: 요청/응답 객체에 접근하는 유틸리티. HTTP뿐 아니라 마이크로서비스·WebSocket 컨텍스트에서도 동작(그래서 추상화됨). `host.switchToHttp()`로 HTTP 컨텍스트를 얻는다.

---

# 5. 필터 바인딩과 스코프

`@UseFilters()`로 필터를 적용합니다. **인스턴스보다 클래스로 넘기는 것을 권장**(DI 지원 + 인스턴스 재사용으로 메모리 절약).

| 스코프 | 방법 |
|--------|------|
| 메서드 | 핸들러 위에 `@UseFilters(HttpExceptionFilter)` |
| 컨트롤러 | 클래스 위에 `@UseFilters(HttpExceptionFilter)` |
| 전역 | `main.ts`에서 `app.useGlobalFilters(new HttpExceptionFilter())` |

```typescript
// 메서드 스코프
@Post()
@UseFilters(HttpExceptionFilter)
async create() { throw new ForbiddenException(); }
```

## 전역 필터 + 의존성 주입

`useGlobalFilters()`로 등록한 전역 필터는 모듈 바깥이라 **DI를 못 씁니다**. DI가 필요하면 `APP_FILTER` 토큰으로 모듈에서 등록합니다.

```typescript
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    { provide: APP_FILTER, useClass: HttpExceptionFilter },
  ],
})
export class AppModule {}
```

---

# 6. 모든 예외 잡기 & 기본 필터 상속

## 모든 예외 잡기

`@Catch()`의 인자를 비우면 타입과 무관하게 **모든** 예외를 잡습니다. `HttpAdapter`를 쓰면 플랫폼 독립적으로 응답할 수 있습니다.

```typescript
@Catch()
export class CatchEverythingFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}
  catch(exception: unknown, host: ArgumentsHost): void {
    const { httpAdapter } = this.httpAdapterHost;
    const ctx = host.switchToHttp();
    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;
    httpAdapter.reply(ctx.getResponse(), {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    }, httpStatus);
  }
}
```

> ⚠️ "모두 잡는 필터"와 "특정 타입 필터"를 함께 쓸 땐, 특정 필터가 제 타입을 처리하도록 **모두 잡는 필터를 먼저** 선언한다.

## 기본 필터 상속

기본 전역 필터의 동작을 재사용하려면 `BaseExceptionFilter`를 상속하고 `super.catch()`를 호출합니다.

```typescript
@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

> ⚠️ `BaseExceptionFilter`를 상속한 메서드·컨트롤러 스코프 필터는 `new`로 만들지 말고 프레임워크가 인스턴스화하게 둔다.

---

# 7. 한눈에 정리

- Nest는 처리되지 않은 예외를 잡는 **내장 예외 계층**을 갖고 있고, 인식 못 하면 500으로 응답한다.
- 오류 상황에선 `HttpException`(또는 `NotFoundException` 등 **내장 예외**)을 던지면 표준 HTTP 응답으로 변환된다.
- 커스텀 예외는 `HttpException`을 **상속**해서 만든다.
- 응답 형식·로깅을 직접 제어하려면 **예외 필터**(`@Catch` + `catch()`)를 만들고 `@UseFilters`로 바인딩한다.
- 스코프는 메서드/컨트롤러/전역. 전역 필터에서 DI가 필요하면 **`APP_FILTER`** 토큰으로 모듈에 등록한다.
- `@Catch()`(빈 인자)로 모든 예외를, `BaseExceptionFilter` 상속으로 기본 동작 확장을 할 수 있다.
