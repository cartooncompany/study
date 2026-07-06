# 모듈(Module)이란 무엇인가

**모듈(Module)**은 `@Module()` 데코레이터가 붙은 클래스로, 관련된 컨트롤러와 프로바이더를 하나로 묶어 **캡슐화**하는 조직 단위입니다. Nest는 이 메타데이터를 이용해 애플리케이션 구조를 구성하고 관리합니다.

모든 Nest 앱에는 최소한 하나의 모듈, 즉 **루트 모듈(root module)**이 있으며, 이는 Nest가 **애플리케이션 그래프**를 구축하는 출발점입니다. 애플리케이션 그래프는 모듈·프로바이더 간의 관계와 의존성을 해결하는 내부 구조입니다.

> 참고: https://docs.nestjs.com/modules

---

# 1. `@Module()`의 네 가지 속성

| 속성 | 설명 |
|------|------|
| `providers` | 이 모듈에서 인스턴스화·공유될 프로바이더 |
| `controllers` | 인스턴스화할 컨트롤러 |
| `imports` | 가져올 다른 모듈들 (그 모듈이 `exports`한 것을 쓰기 위해) |
| `exports` | 이 모듈이 외부에 공개할 프로바이더 (모듈의 public API) |

모듈은 기본적으로 프로바이더를 **캡슐화**합니다. 즉 내 모듈에 속한 프로바이더이거나, 임포트한 모듈이 명시적으로 `exports`한 프로바이더만 주입할 수 있습니다.

---

# 2. 기능 모듈(Feature module)

관련 코드를 도메인 단위로 묶은 모듈입니다. 경계가 명확해지고 SOLID 원칙에도 부합합니다.

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

루트 모듈에서 임포트해 앱에 연결합니다.

```typescript
@Module({ imports: [CatsModule] })
export class AppModule {}
```

---

# 3. 공유 모듈(Shared module)

모듈은 **싱글턴**이므로, `exports`한 프로바이더는 이를 임포트하는 여러 모듈에서 **같은 인스턴스**로 공유됩니다.

```typescript
@Module({
  providers: [CatsService],
  exports: [CatsService],   // ← 공유하려면 export 필수
})
export class CatsModule {}
```

만약 필요한 모듈마다 `CatsService`를 직접 등록하면, 모듈마다 **별개의 인스턴스**가 생깁니다. 이는 메모리 사용을 늘리고, 서비스가 내부 상태를 가질 경우 상태 불일치 같은 예기치 않은 동작을 유발할 수 있습니다. 하나의 모듈에서 캡슐화하고 `exports`하면 같은 인스턴스가 재사용되어 더 예측 가능해집니다.

---

# 4. 모듈 재내보내기(Re-exporting)

임포트한 모듈을 다시 `exports`하면, 이 모듈을 가져오는 쪽에서도 그 모듈을 쓸 수 있습니다.

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

---

# 5. 전역 모듈(Global module)

`@Global()`을 붙이면 임포트 없이 어디서나 프로바이더를 쓸 수 있습니다. 전역 모듈은 루트나 코어 모듈에서 **한 번만** 등록해야 합니다.

```typescript
@Global()
@Module({
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

다만 남용은 권장되지 않습니다. 가능하면 `imports` 배열로 명시적으로 공유하는 편이 구조와 유지보수 면에서 낫고, 불필요한 결합을 피할 수 있습니다.

---

# 6. 동적 모듈(Dynamic module)

런타임에 설정 가능한 모듈입니다. `forRoot()` 같은 정적 메서드가 `DynamicModule`을 반환하며, 전달된 옵션에 따라 프로바이더를 만들어 노출합니다.

```typescript
@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers,
      exports: providers,
    };
  }
}
```

- 반환한 속성들은 `@Module()`에 정의된 기본 메타데이터를 **덮어쓰지 않고 확장(extend)**합니다.
- 전역으로 만들려면 반환 객체에 `global: true`를 둡니다.
- 사용: `imports: [DatabaseModule.forRoot([User])]`

---

# 7. 참고: 모듈에도 주입할 수 있다

모듈 클래스도 생성자로 프로바이더를 주입받을 수 있습니다(설정 목적 등). 단, **모듈 클래스 자체는 프로바이더로 주입될 수 없습니다**(순환 의존성 때문).

---

# 8. 한눈에 정리

- 모듈은 `@Module()`이 붙은 클래스이고, 모든 앱엔 최소 하나의 **루트 모듈**이 있다.
- 네 속성: `providers` / `controllers` / `imports` / `exports`.
- 모듈은 프로바이더를 **캡슐화**하므로, `exports`한 것만 다른 모듈에서 쓸 수 있다.
- 모듈은 **싱글턴**이라 `exports`한 프로바이더는 **같은 인스턴스**로 공유된다.
- `@Global()`로 전역 모듈을, `forRoot()`로 **동적 모듈**을 만들 수 있다(전역 남용은 지양).
