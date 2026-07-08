# 가드(Guard)란 무엇인가

**가드(Guard)**는 `@Injectable()`이 붙고 `CanActivate` 인터페이스를 구현한 클래스입니다. 단일 책임은 **인가(authorization)** — 즉 현재 요청이 라우트 핸들러까지 도달해도 되는지를 런타임 조건(권한·역할·토큰 등)에 따라 결정합니다.

가드는 미들웨어와 자주 비교됩니다. 미들웨어로도 인증을 할 수 있지만, 미들웨어는 **다음에 어떤 핸들러가 실행되는지 모릅니다**(`next()`만 호출). 반면 가드는 `ExecutionContext`에 접근할 수 있어 **무엇이 다음에 실행될지 정확히 알고** 판단할 수 있습니다.

**실행 순서:** 모든 **미들웨어 이후**, 그리고 어떤 **인터셉터·파이프보다 먼저** 실행됩니다. (→ [[middleware]], [[pipes]])

> 참고: https://docs.nestjs.com/guards

---

# 1. CanActivate 인터페이스

모든 가드는 `canActivate()` 메서드를 구현합니다. 반환값이 판정입니다.

```typescript
canActivate(
  context: ExecutionContext,
): boolean | Promise<boolean> | Observable<boolean>;
```

- `true` → 요청 통과, 핸들러 실행.
- `false` → 요청 거부. Nest가 자동으로 **403 Forbidden**으로 응답합니다.
- 동기(`boolean`)뿐 아니라 비동기(`Promise`)·`Observable`도 반환 가능.

---

# 2. 기본 예: AuthGuard

`ExecutionContext`에서 요청 객체를 꺼내 검증 함수에 넘기는 형태입니다.

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

## ExecutionContext

`ExecutionContext`는 `ArgumentsHost`(예외 필터에서 봤던 그것 → [[exception-filters]])를 **상속**해, 현재 실행 과정에 대한 추가 정보를 제공합니다.

- `context.switchToHttp().getRequest()` — HTTP 요청 객체 획득.
- `context.getHandler()` — 곧 실행될 **핸들러(메서드)** 참조. (메타데이터 조회에 사용)
- `context.getClass()` — 해당 **컨트롤러 클래스** 참조.

---

# 3. 가드 바인딩

`@UseGuards()` 데코레이터로 붙입니다. 예외 필터·파이프와 마찬가지로 **인스턴스보다 클래스로** 넘기는 것이 권장됩니다(DI 지원 + 인스턴스 재사용).

| 스코프 | 방법 |
|--------|------|
| 메서드 | 핸들러 위에 `@UseGuards(AuthGuard)` |
| 컨트롤러 | 클래스 위에 `@UseGuards(RolesGuard)` |
| 전역 | `main.ts`에서 `app.useGlobalGuards(new RolesGuard())` |

```typescript
// 컨트롤러 스코프 — 이 컨트롤러의 모든 핸들러에 적용
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

## 전역 가드 + 의존성 주입

`useGlobalGuards()`로 등록한 전역 가드는 모듈 바깥이라 **DI를 못 씁니다**. DI(예: `Reflector`, 서비스 주입)가 필요하면 `APP_GUARD` 토큰으로 모듈에서 등록합니다. (→ [[exception-filters]]의 `APP_FILTER`, [[pipes]]의 `APP_PIPE`와 동일 패턴)

```typescript
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}
```

---

# 4. 역할 기반 인가 (Role-based)

핸들러마다 "이 엔드포인트는 어떤 역할이 필요한가"를 **메타데이터**로 붙여 두고, 가드가 그 메타데이터를 읽어 사용자 역할과 비교하는 패턴입니다.

## 4-1. 커스텀 데코레이터 만들기

`Reflector.createDecorator`로 타입 안전한 메타데이터 데코레이터를 만듭니다.

```typescript
// roles.decorator.ts
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

## 4-2. 핸들러에 붙이기

```typescript
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

## 4-3. RolesGuard에서 메타데이터 읽기

`Reflector`를 주입받아 `reflector.get(Roles, context.getHandler())`로 그 핸들러에 붙은 역할 정보를 꺼냅니다.

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true; // 역할 조건이 없는 핸들러는 그냥 통과
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

- `roles`가 없으면(= `@Roles`가 안 붙은 핸들러) 제한 없이 통과.
- `matchRoles`는 필요한 역할과 사용자 역할을 비교하는 사용자 정의 함수.
- `user`는 보통 앞선 인증 가드/미들웨어가 요청에 심어 둔 값.

---

# 5. 거부 동작

가드가 `false`를 반환하면 Nest가 자동으로 아래 403 응답을 보냅니다(내부적으로 `ForbiddenException`을 던짐).

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

다른 응답을 주고 싶으면 가드 안에서 직접 예외를 던지면 됩니다(예: `UnauthorizedException`). 그 예외는 예외 필터 계층이 처리합니다. (→ [[exception-filters]])

---

# 6. 한눈에 정리

- 가드는 `CanActivate`를 구현한 클래스로, **인가**(요청을 핸들러까지 보낼지)를 담당한다.
- `canActivate(context)`가 `true`면 통과, `false`면 자동 **403**(`ForbiddenException`).
- 미들웨어와 달리 **`ExecutionContext`** 를 통해 다음에 실행될 핸들러/클래스를 알 수 있다. 실행 순서는 **미들웨어 이후, 인터셉터·파이프 이전**.
- `@UseGuards()`로 메서드/컨트롤러/전역에 바인딩. 전역 가드에서 DI가 필요하면 **`APP_GUARD`** 토큰으로 모듈에 등록.
- 역할 기반 인가는 `Reflector.createDecorator`로 만든 **`@Roles`** 메타데이터 + 가드의 **`Reflector`** 조회(`reflector.get(Roles, context.getHandler())`)로 구현한다.

## ✅ 셀프 체크

1. 가드와 미들웨어의 결정적 차이는? (힌트: 아는 정보의 양)
2. 가드는 인터셉터·파이프·미들웨어 중 무엇의 앞/뒤에 실행되나?
3. `canActivate`가 `false`를 반환하면 클라이언트는 어떤 응답을 받나?
4. 전역 가드에서 `Reflector`를 주입받으려면 어떻게 등록해야 하나?
5. `RolesGuard`에서 `context.getHandler()`를 쓰는 이유는?
