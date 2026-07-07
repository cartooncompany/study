# 파이프(Pipe)란 무엇인가

파이프는 `PipeTransform` 인터페이스를 구현한 **주입 가능한 클래스**입니다. 컨트롤러의 라우트 핸들러가 실행되기 **직전**에, 그 핸들러가 받을 인자(argument)를 가로채서 처리합니다. 주로 두 가지 용도로 씁니다.

- **변환(transformation)**: 입력 데이터를 원하는 형태로 바꾼다. (예: 문자열 `"3"` → 숫자 `3`)
- **검증(validation)**: 입력이 유효하면 그대로 통과, 유효하지 않으면 예외를 던진다.

파이프는 **예외 존(exceptions zone)** 안에서 동작합니다. 파이프에서 예외를 던지면 라우트 핸들러는 실행되지 않고, 전역 예외 필터가 그 예외를 처리합니다. (→ [[exception-filters]])

> 참고: https://docs.nestjs.com/pipes

---

# 1. 내장 파이프

`@nestjs/common`에 바로 쓸 수 있는 파이프 10종이 들어 있습니다.

| 파이프 | 역할 |
|--------|------|
| `ValidationPipe` | 객체 전체를 class-validator 규칙으로 검증 |
| `ParseIntPipe` | 문자열 → 정수 |
| `ParseFloatPipe` | 문자열 → 실수 |
| `ParseBoolPipe` | 문자열 → 불리언 |
| `ParseArrayPipe` | 문자열 → 배열 |
| `ParseUUIDPipe` | UUID 형식 검증 |
| `ParseEnumPipe` | 열거형(enum) 값 검증 |
| `DefaultValuePipe` | 값이 없을 때 기본값 제공 |
| `ParseFilePipe` | 업로드 파일 검증 |
| `ParseDatePipe` | 문자열 → Date |

`Parse*` 파이프들은 이름 그대로 "변환 + 실패 시 검증 예외"를 함께 수행합니다.

---

# 2. 파이프 바인딩

## 파라미터 스코프 — 특정 인자에만 적용

`@Param('id', ParseIntPipe)`처럼 데코레이터 두 번째 인자로 넘깁니다. `id`는 핸들러에 도달하기 전에 반드시 숫자가 되거나 예외가 발생합니다.

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

**클래스**를 넘기면 Nest가 인스턴스화·DI를 담당합니다. 반면 옵션을 바꿔야 하면 **인스턴스**를 직접 만들어 넘깁니다.

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

## 메서드 스코프 — `@UsePipes()`

```typescript
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

## 전역 스코프

```typescript
// main.ts
app.useGlobalPipes(new ValidationPipe());
```

---

# 3. 커스텀 파이프

`PipeTransform` 인터페이스의 `transform` 메서드만 구현하면 됩니다.

```typescript
interface PipeTransform<T, R> {
  transform(value: T, metadata: ArgumentMetadata): R;
}
```

- **`value`**: 처리 대상인 실제 인자 값.
- **`metadata`**: 그 인자에 대한 부가 정보.

`ArgumentMetadata`의 필드:

| 필드 | 의미 |
|------|------|
| `type` | 인자 출처. `'body'` · `'query'` · `'param'` · `'custom'` |
| `metatype` | 인자의 타입(예: `CreateCatDto`). 라우트 핸들러에 타입 표기가 없거나 순수 JS면 `undefined` |
| `data` | 데코레이터에 넘긴 문자열(예: `@Body('name')`의 `'name'`) |

## 예: ParseIntPipe 직접 구현

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

`transform`이 반환하는 값이 그대로 핸들러 인자로 전달됩니다.

---

# 4. 검증 파이프 만들기 (핵심)

검증 로직을 라우트 핸들러 밖으로 빼면 컨트롤러가 단일 책임을 지키게 됩니다. 두 가지 흔한 방식이 있습니다.

## 방식 A — 스키마 기반 (Zod)

스키마를 주입받아 `parse`로 검증합니다. 실패 시 `BadRequestException`을 던집니다.

```typescript
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      return this.schema.parse(value);
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

스키마마다 인스턴스가 달라지므로 `@UsePipes(new ZodValidationPipe(createCatSchema))`처럼 **인스턴스**로 바인딩합니다.

## 방식 B — 데코레이터 기반 (class-validator)

DTO 클래스에 `@IsString()` 같은 데코레이터로 규칙을 선언해 두고, 파이프가 그 규칙으로 검증합니다.

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

포인트:

- **`plainToInstance`**: 요청 본문(순수 객체)을 DTO 클래스 인스턴스로 변환해야 class-validator가 데코레이터 규칙을 읽을 수 있음.
- **`toValidate`**: `metatype`이 `String`·`Number` 같은 원시 타입이면 검증 대상이 아니므로 건너뜀.
- 여기서는 `value`를 그대로 반환(검증만). 변환까지 하려면 `object`를 반환.

---

# 5. 전역 검증 파이프 + DI

전역 파이프는 보통 앱 전체에 검증을 한 번에 걸 때 씁니다.

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
```

단, `useGlobalPipes`로 등록한 파이프는 모듈 바깥이라 **DI를 쓸 수 없습니다**. 의존성 주입이 필요하면 예외 필터와 마찬가지로 토큰(`APP_PIPE`)으로 모듈에서 등록합니다. (→ [[exception-filters]]의 `APP_FILTER`와 동일한 패턴)

```typescript
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

---

# 6. 한눈에 정리

- 파이프는 `PipeTransform`을 구현한 클래스로, 핸들러 **실행 직전**에 인자를 **변환**하거나 **검증**한다.
- `transform(value, metadata)`가 반환한 값이 그대로 핸들러 인자가 된다. 검증 실패 시 예외를 던지면 예외 필터가 처리한다.
- `ParseIntPipe` 등 **내장 파이프 10종**을 바로 쓸 수 있고, 옵션이 필요하면 `new ParseIntPipe({...})`로 인스턴스를 넘긴다.
- 바인딩 스코프: **파라미터**(`@Param('id', Pipe)`) / **메서드**(`@UsePipes`) / **전역**(`useGlobalPipes`).
- 검증 파이프는 **스키마 기반(Zod)** 또는 **데코레이터 기반(class-validator + `metatype`)** 으로 만든다.
- 전역 파이프에서 DI가 필요하면 **`APP_PIPE`** 토큰으로 모듈에 등록한다.

## ✅ 셀프 체크

1. 파이프의 두 가지 주 용도는? 파이프가 예외를 던지면 핸들러는 어떻게 되나?
2. `ParseIntPipe`를 **클래스**로 넘길 때와 **인스턴스**로 넘길 때의 차이는?
3. `ArgumentMetadata`의 `metatype`은 어떤 값이고, class-validator 파이프에서 왜 필요한가?
4. class-validator 검증 전에 `plainToInstance`를 호출하는 이유는?
5. 전역 파이프에서 의존성 주입을 쓰려면 어떻게 등록해야 하나?
