# 커스텀 데코레이터(Custom Route Decorator)란 무엇인가

Nest는 **데코레이터** 위에 지어진 프레임워크입니다. `@Get()`, `@Body()`, `@UseGuards()` 처럼 많은 기능을 데코레이터로 표현하죠. 이 문서는 그중 **매개변수 데코레이터(param decorator)** 를 직접 만드는 방법을 다룹니다.

## 내장 매개변수 데코레이터

Nest는 순수 Express/Fastify 객체를 편하게 꺼내 쓰도록 다음 데코레이터를 기본 제공합니다.

| 데코레이터 | Express 대응 |
|-----------|-------------|
| `@Request()`, `@Req()` | `req` |
| `@Response()`, `@Res()` | `res` |
| `@Next()` | `next` |
| `@Session()` | `req.session` |
| `@Param(key?)` | `req.params` / `req.params[key]` |
| `@Body(key?)` | `req.body` / `req.body[key]` |
| `@Query(key?)` | `req.query` / `req.query[key]` |
| `@Headers(key?)` | `req.headers` / `req.headers[key]` |
| `@Ip()` | `req.ip` |
| `@HostParam()` | `req.hosts` |

> 참고: https://docs.nestjs.com/custom-decorators

---

# 1. 커스텀 매개변수 데코레이터 만들기

`createParamDecorator`를 쓰면 나만의 매개변수 데코레이터를 만들 수 있습니다. 예를 들어 인증 미들웨어/가드가 `request.user`에 심어둔 사용자 객체를 매번 `req.user`로 꺼내는 대신, `@User()`로 바로 주입받을 수 있습니다.

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

- 콜백은 `(data, ctx)`를 받는다. `ctx`는 `ExecutionContext`(가드·인터셉터에서 봤던 그것 → [[07-guards]], [[08-interceptors]]).
- `ctx.switchToHttp().getRequest()`로 요청을 꺼내고, 반환한 값이 그대로 핸들러 인자로 주입된다.

## 사용

```typescript
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
```

---

# 2. 데이터 전달 — 특정 속성만 뽑기

데코레이터에 인자를 넘기면 콜백의 첫 번째 매개변수 `data`로 들어옵니다. 이를 이용해 사용자 객체의 특정 속성만 꺼낼 수 있습니다.

```typescript
export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
```

## 사용

```typescript
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

`@User()` → 사용자 객체 전체, `@User('firstName')` → `user.firstName`만 주입됩니다.

---

# 3. 파이프와 함께 쓰기

커스텀 매개변수 데코레이터에도 **파이프**를 적용할 수 있습니다. 검증·변환을 그대로 얹을 수 있다는 뜻입니다. (→ [[06-pipes]])

```typescript
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
```

> 참고: 커스텀 데코레이터에 `ValidationPipe`를 적용하려면 `validateCustomDecorators: true` 옵션이 필요합니다(기본값은 커스텀 데코레이터 검증을 건너뜀).

---

# 4. 데코레이터 합성 (applyDecorators)

여러 데코레이터를 하나로 묶어 재사용할 수 있습니다. 예를 들어 인증 관련 데코레이터 4개를 매번 나열하는 대신 `@Auth()` 하나로 합칩니다.

```typescript
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

## 사용

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

`@Auth('admin')` 한 줄이 위 4개 데코레이터를 모두 적용한 것과 같습니다. 중복을 줄이고 의도를 명확하게 드러냅니다.

---

# 5. 한눈에 정리

- Nest는 데코레이터 기반이며, `@Param`·`@Body`·`@Query` 같은 **내장 매개변수 데코레이터**로 요청 객체를 편하게 꺼낸다.
- **`createParamDecorator`** 로 나만의 매개변수 데코레이터를 만든다. 콜백 `(data, ctx)`가 반환한 값이 핸들러 인자로 주입된다.
- 데코레이터에 넘긴 인자는 콜백의 **`data`** 로 들어와, `@User('firstName')`처럼 특정 속성만 뽑을 수 있다.
- 커스텀 매개변수 데코레이터에도 **파이프**를 적용할 수 있다(`ValidationPipe`는 `validateCustomDecorators: true` 필요).
- **`applyDecorators`** 로 여러 데코레이터를 하나로 **합성**해 재사용성을 높인다.

## ✅ 셀프 체크

1. `createParamDecorator` 콜백의 두 매개변수는 무엇이고, 반환값은 어디로 가나?
2. `@User('email')`을 만들려면 콜백에서 `data`를 어떻게 활용하나?
3. 커스텀 데코레이터에 `ValidationPipe`를 적용할 때 꼭 필요한 옵션은?
4. `applyDecorators`는 어떤 문제를 해결하나?
5. `ctx.switchToHttp().getRequest()`에서 `ctx`의 타입은? 이 타입을 다른 어디서 봤나?
