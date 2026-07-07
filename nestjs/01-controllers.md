# 컨트롤러(Controller)란 무엇인가

**컨트롤러(Controller)**는 들어오는 **요청(request)**을 받아 처리하고 클라이언트에 **응답(response)**을 돌려주는 진입점입니다.

어떤 컨트롤러가 각 요청을 처리할지는 **라우팅(routing)** 메커니즘이 결정합니다. 하나의 컨트롤러는 보통 여러 라우트를 가지며, 각 라우트는 서로 다른 동작을 수행합니다. 컨트롤러는 클래스와 **데코레이터(decorator)**로 정의하며, 데코레이터가 클래스에 메타데이터를 붙여 Nest가 요청과 핸들러를 연결하는 라우팅 맵을 만들 수 있게 합니다.

> 참고: https://docs.nestjs.com/controllers

---

# 1. 라우팅: 경로는 어떻게 정해지는가

`@Controller()` 데코레이터는 컨트롤러를 정의하는 **필수** 요소입니다. 인자로 경로 접두사(prefix)를 줄 수 있고, 이 접두사는 관련 라우트를 묶고 중복 경로를 줄여 줍니다.

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

최종 라우트 경로는 다음 두 가지의 조합으로 결정됩니다.

**컨트롤러 접두사 + 메서드 데코레이터 경로**

- `@Controller('cats')` + `@Get()` → `GET /cats`
- `@Controller('cats')` + `@Get('breed')` → `GET /cats/breed`

메서드 이름(`findAll`)은 임의로 정해도 됩니다. Nest는 라우트를 바인딩할 메서드가 필요할 뿐, 그 이름 자체에는 의미를 두지 않습니다.

HTTP 메서드마다 데코레이터가 준비되어 있습니다.

`@Get()` `@Post()` `@Put()` `@Delete()` `@Patch()` `@Options()` `@Head()`, 그리고 이 전부를 처리하는 `@All()`.

---

# 2. 응답을 다루는 두 가지 방식

Nest는 응답을 조작하는 데 서로 다른 두 방식을 제공합니다.

## 2.1 표준 방식 (권장)

핸들러가 **객체나 배열**을 반환하면 **자동으로 JSON 직렬화**됩니다. `string`, `number`, `boolean` 같은 원시 타입은 직렬화 없이 값만 그대로 전송됩니다. 즉 값을 `return`만 하면 나머지는 Nest가 처리합니다.

- 기본 상태 코드는 항상 **200** (단, POST 요청은 **201**)
- `@HttpCode(...)` 데코레이터로 변경 가능

## 2.2 라이브러리 특화 방식

`@Res()` 데코레이터로 Express 같은 하부 플랫폼의 **응답 객체**를 직접 주입받아 `res.status(200).send()`처럼 네이티브 메서드를 씁니다.

```typescript
@Get()
findAll(@Res() res: Response) {
  res.status(HttpStatus.OK).json([]);
}
```

주의할 점:

- `@Res()`나 `@Next()`를 쓰면 그 라우트는 표준 방식이 **자동으로 꺼집니다**. 이때는 직접 응답을 보내지 않으면 서버가 멈춘 채 걸립니다.
- 두 방식을 함께 쓰려면(예: 쿠키/헤더만 손대고 나머지는 프레임워크에 맡김) `@Res({ passthrough: true })`를 사용합니다.
- 라이브러리 특화 방식은 플랫폼 의존적이 되고 테스트가 어려워지며, 인터셉터나 `@HttpCode()`/`@Header()` 같은 표준 기능과의 호환성을 잃으므로 신중히 써야 합니다.

---

# 3. 요청 데이터 추출하기

요청 객체 전체는 `@Req()`로 주입받을 수 있지만, 대부분은 전용 데코레이터를 쓰는 편이 깔끔합니다.

| 데코레이터 | 대응 값 | 용도 |
|-----------|---------|------|
| `@Req()` | `req` | 요청 객체 전체 |
| `@Body()` | `req.body` | 요청 본문 |
| `@Query('age')` | `req.query.age` | 쿼리 파라미터 (`?age=2`) |
| `@Param('id')` | `req.params.id` | 라우트 파라미터 (`/:id`) |
| `@Headers()` | `req.headers` | 헤더 |
| `@Ip()` | `req.ip` | 클라이언트 IP |

## 3.1 라우트 파라미터

동적 값(`GET /cats/1`)을 받으려면 경로에 토큰(`:id`)을 넣고 `@Param()`으로 꺼냅니다.

```typescript
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
```

주의: **매개변수 라우트(`:id`)는 정적 경로보다 뒤에 선언**해야 합니다. 그렇지 않으면 매개변수 경로가 정적 경로로 가야 할 요청까지 가로챕니다.

## 3.2 쿼리 파라미터

```typescript
@Get()
findAll(@Query('age') age: number, @Query('breed') breed: string) { ... }
// GET /cats?age=2&breed=Persian  →  age=2, breed='Persian'
```

중첩 객체나 배열 같은 복잡한 쿼리는 HTTP 어댑터의 파서 설정이 필요합니다(Express는 `extended` 파서).

---

# 4. DTO: 요청 본문의 형태 정의

**DTO(Data Transfer Object)**는 네트워크로 데이터가 어떻게 전송되는지 명시하는 객체입니다.

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

여기서 중요한 점은 **인터페이스가 아니라 클래스로 정의**한다는 것입니다.

- 인터페이스는 트랜스파일 과정에서 사라져 런타임에 참조할 수 없습니다.
- 클래스는 ES6 표준이라 컴파일된 JS에도 실체로 남습니다. 그래서 **파이프(Pipe)** 같은 기능이 런타임에 메타타입을 참조할 수 있습니다.

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) { ... }
```

---

# 5. 그 밖의 유용한 기능

- **와일드카드**: `@Get('abcd/*')` → `abcd/`로 시작하는 모든 경로 매칭.
- **상태 코드 변경**: `@HttpCode(204)`
- **응답 헤더 추가**: `@Header('Cache-Control', 'no-store')`
- **리다이렉트**: `@Redirect('https://nestjs.com', 301)` (기본 302). 반환 객체로 URL·코드를 동적으로 덮어쓸 수 있음.
- **서브도메인 라우팅**: `@Controller({ host: ':account.example.com' })` + `@HostParam('account')`
- **비동기**: `async` + `Promise` 반환 가능, RxJS `Observable` 반환도 지원(Nest가 구독을 대신 처리).

---

# 6. 컨트롤러 등록

컨트롤러는 반드시 모듈의 `controllers` 배열에 등록해야 Nest가 인스턴스를 만들고 라우트를 연결합니다.

```typescript
@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

---

# 7. 한눈에 정리

- 컨트롤러는 **요청을 받아 응답을 돌려주는 입구**이며 `@Controller()` + HTTP 메서드 데코레이터로 정의한다.
- 최종 경로 = **접두사 + 메서드 경로**.
- 응답은 **표준 방식(값 return → 자동 JSON, 200/201)**을 기본으로 하고, `@Res()`는 필요할 때만.
- 요청 데이터는 `@Body()` `@Query()` `@Param()` 등 **전용 데코레이터**로 꺼낸다.
- 요청 본문 형태는 **DTO 클래스**로 정의한다(인터페이스 아님).
- 컨트롤러는 **모듈의 `controllers`에 등록**해야 동작한다.
