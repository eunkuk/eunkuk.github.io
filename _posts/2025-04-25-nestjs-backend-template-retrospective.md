---
title: "NestJS 백엔드 보일러플레이트를 만들었다 — 매번 반복하는 기반 작업을 모듈로 묶기까지"
date: 2025-04-25 22:00:00 +0900
categories: [Dev, Backend]
tags: [nestjs, typescript, boilerplate, jwt, redis, swagger, typeorm, winston, s3, architecture, retrospective]
description: "Java/Spring Boot 개발자가 NestJS를 찍먹하며 만든 백엔드 보일러플레이트. 인증/예외처리/로깅/파일업로드를 모듈로 묶고, Spring Boot와 비교하며 느낀 점을 정리한다."
---

## 1. 왜 NestJS를 찍먹했나

본업은 Java/Spring Boot다. 평소에 하는 일도 Spring Boot + JPA + QueryDSL 조합이고, 익숙한 환경에서 벗어날 이유가 딱히 없었다.

그런데 Node.js 쪽 프로젝트를 잠깐 맡게 되면서, "Spring Boot 개발자가 가장 적응하기 쉬운 Node 프레임워크가 뭐지?" 찾다가 NestJS를 알게 됐다. 모듈, DI, 데코레이터, Guard, Interceptor — 용어부터 Spring이랑 비슷해서 진입장벽이 낮았다.

본격적으로 프로덕션에 쓸 생각은 아니었고, **"Spring Boot에서 매번 하던 기반 작업을 NestJS로 하면 어떤 느낌일까?"** 정도로 시작했다. 그래서 만든 게 이 보일러플레이트다.

- JWT 인증 + Redis refresh token 저장
- 전역 예외 필터 + 응답 포맷 통일
- Winston 로깅 + 요청 traceId
- Swagger 문서화 + BearerAuth 설정
- 파일 업로드 (로컬 / S3)
- 환경변수 분리 (`.env.${NODE_ENV}`)

Spring Boot에서 매번 세팅하는 것들을 NestJS로 옮겨본 셈이다.

> [eunkuk/nest-test](https://github.com/eunkuk/nest-test)

---

## 2. 구조: 빼기 쉬운 모듈

```
src/
├── auth/          # JWT 인증 (register/login/refresh/logout)
├── aws/s3/        # S3 업로드/다운로드
├── config/        # 앱 설정
├── constants/     # 에러 코드, 파일 상수
├── database/      # TypeORM + PostgreSQL
├── entities/      # User, Exam, Config + seed 스크립트
├── exam/          # 샘플 CRUD (참고용)
├── files/         # 로컬/S3 파일 업로드
├── filters/       # 전역 예외 필터
├── guards/        # JWT AuthGuard
├── http/          # Axios 래핑 (외부 API 호출)
├── interceptor/   # 로깅 인터셉터
├── lib/           # 공통 유틸 (응답, 페이징, 문서 데코레이터, 예외)
├── logging/       # Winston 설정
├── mailer/        # AWS SES + Handlebars 보일러플레이트
├── middleware/     # favicon 무시 등
├── parser/        # RSS 파서, 웹 크롤러 (cheerio)
├── pipes/         # 파일 검증 파이프
├── redis/         # ioredis 연결/서비스
├── schedule/      # cron 잡 뼈대
└── templates/     # 메일 보일러플레이트 (.hbs)
```

`AppModule`의 imports에 12개 모듈이 나열되어 있다. 핵심은 **빼기 쉬운가**였다.

메일 기능이 필요 없으면 `MailerModule` import를 지운다. S3 대신 로컬만 쓸 거면 `S3Module`을 뺀다. Parser를 별도 서비스로 분리하고 싶으면 폴더째 옮기면 된다. 보일러플레이트에서 중요한 건 기능이 많은 게 아니라 **불필요한 걸 뺄 때 다른 데가 안 깨지는 것**이다.

---

## 3. 예외/응답 표준화

프로젝트가 커지면 응답 포맷이 슬금슬금 흐트러진다. 초반에 틀을 강하게 잡아두고 싶었다.

### 에러 코드 상수

```typescript
export const CODE = {
  OK:                  { status: HttpStatus.OK, code: 'G001', msg: '성공' },
  CREATED:             { status: HttpStatus.CREATED, code: 'G002', msg: '생성 성공' },
  BAD_REQUEST:         { status: HttpStatus.BAD_REQUEST, code: 'C001', msg: '잘못된 요청입니다' },
  UNAUTHORIZED:        { status: HttpStatus.UNAUTHORIZED, code: 'C002', msg: '인증이 필요합니다' },
  TOKEN_EXPIRED:       { status: HttpStatus.UNAUTHORIZED, code: 'C003', msg: '토큰이 만료되었습니다' },
  INVALID_TOKEN:       { status: HttpStatus.UNAUTHORIZED, code: 'C004', msg: '유효하지 않은 토큰입니다' },
  INTERNAL_SERVER_ERROR: { status: HttpStatus.INTERNAL_SERVER_ERROR, code: 'S001', msg: '서버 오류' },
  // ...25개
};
```

`G` = 성공, `C` = 클라이언트 에러, `S` = 서버 에러. HTTP 상태 코드 + 내부 코드 + 메시지를 한 곳에 모았다.

### 통일된 응답 포맷

```typescript
class ResponseEntity<T> {
  private constructor(
    private code: string,
    private message: string,
    private data: T,
  ) {}

  static OK<T>(data?: T, message = 'OK') {
    return new ResponseEntity(CODE.OK, message, data);
  }

  static ERROR(status: IStatus, message?: string) {
    throw new ApiException(status, message);
  }
}
```

성공이면 `ResponseEntity.OK(data)`, 실패면 `throw new ApiException(CODE.XXX)`. 컨트롤러에서 이 규칙만 따르면 전역 필터가 응답을 `{ code, message, data }` 형태로 통일한다.

### 전역 예외 필터

```typescript
@Catch()
export class ApiExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    // ApiException → 에러 코드 그대로
    // HttpException → status 코드로 CODE 매핑
    // 그 외 → INTERNAL_SERVER_ERROR
    response.status(status).json({ code, message, data: null });
  }
}
```

어떤 예외가 터져도 클라이언트는 같은 형태의 응답을 받는다.

---

## 4. 인증: JWT + Redis

### 흐름

```
register → bcrypt 해싱 → User 저장
login    → 비밀번호 비교 → access/refresh 발급 → refresh를 Redis에 저장
refresh  → Redis의 refresh와 비교 → 새 토큰 쌍 발급
logout   → Redis에서 refresh 삭제 + access를 블랙리스트에 등록
```

refresh token은 Redis에 TTL과 함께 저장한다. 로그아웃하면 refresh를 삭제하고, access token은 `blacklist:{email}` 키로 Redis에 넣어 남은 유효시간 동안 차단한다.

### AuthGuard

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const token = this.extractToken(request);

    // 1) JWT 서명 검증
    const payload = this.jwtService.verify(token, { secret });

    // 2) 블랙리스트 확인
    const isBlacklisted = await this.redisService.get(`blacklist:${payload.email}`);
    if (isBlacklisted) throw new ApiException(CODE.UNAUTHORIZED);

    // 3) request에 payload 주입
    request['user'] = payload;
    return true;
  }
}
```

세 단계 검증: 토큰 존재 → 서명 유효성 → 블랙리스트 미등록.

---

## 5. 로깅: traceId로 요청 추적

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const traceId = uuidv4();
    const { method, url } = request;

    this.logger.log(`Start: ${controller} ${handler} ${method} ${url} ${traceId}`);

    return next.handle().pipe(
      tap(() => this.logger.log(`End: ${method} ${url} - ${duration}ms ${traceId}`)),
      catchError(err => { /* 에러 로깅 */ }),
    );
  }
}
```

요청마다 UUID 기반 `traceId`를 생성한다. 시작/종료 로그에 같은 traceId가 붙으니까, 로그가 섞여도 하나의 요청 흐름을 추적할 수 있다.

payload는 `truncateAndStringify()`로 1000자까지만 기록한다. File/Blob은 `"size: {bytes} bytes"`로 대체하고, 배열이 10개 넘으면 `[Array(N)]`으로 축약한다. 로그가 너무 커지는 걸 방지하면서 디버깅에 필요한 정보는 남긴다.

---

## 6. Swagger: 데코레이터로 문서화 캡슐화

Swagger 문서화는 처음엔 열심히 하다가 금방 방치된다. `@ApiOperation`, `@ApiResponse`, `@ApiBody`를 매번 붙이는 게 귀찮아서다.

그래서 커스텀 데코레이터로 묶었다.

```typescript
// 사용 예: 파일 업로드 API 문서화
@FileS3Post()  // 이 한 줄로:
               // - multipart/form-data consumes 설정
               // - FilesInterceptor 적용
               // - Swagger에 binary 배열 스키마 반영
               // - 인증 필요 표시 + 에러 응답 문서화
```

개별 API에 데코레이터 5~6개를 나열하는 대신, 도메인별 커스텀 데코레이터 하나로 끝난다. 쓰기 쉬워지면 방치될 확률이 줄어든다.

---

## 7. 파일 업로드: 로컬과 S3 둘 다

초기 개발은 로컬 저장으로 시작했다가 운영에서 S3로 옮기는 경우가 흔하다. 둘 다 넣어뒀다.

```
POST /files/upload-local  → 서버 uploads/ 폴더
POST /files/upload-s3     → S3 temp/ 경로
GET  /files/local/:name   → 로컬 stream
GET  /files/s3/:name      → S3 GetObject stream
```

`FileValidationPipe`로 기본 안전장치를 건다.

```typescript
// 최대 3개, 5MB 제한, 허용 mime 타입만
validate(files: Express.Multer.File[]) {
  if (files.length > 3) throw ...;
  for (const file of files) {
    if (file.size > 5 * 1024 * 1024) throw ...;
    if (!ALLOWED_MIME_TYPES.includes(file.mimetype)) throw ...;
  }
}
```

---

## 8. 돌아보며

### 잘한 것

1. **모듈 경계가 명확하다.** 기능을 빼고 끼우기 쉽다. 새 프로젝트에서 필요 없는 모듈을 import에서 지우면 끝이다.
2. **예외/응답/로깅을 전역 레벨에서 통일했다.** 이후 기능이 추가돼도 규칙이 흔들리지 않는다.
3. **Swagger 데코레이터 캡슐화.** 문서화가 귀찮지 않으니까 유지된다.

### 아쉬운 것

1. **AuthGuard의 payload 주입이 어긋난다.** `request['user'] = payload.user`로 주입하는데, 실제 payload에는 `{ email, role }` 형태라 `payload.user`는 `undefined`다. 그냥 `payload`를 넣어야 한다.
2. **로그아웃 시 블랙리스트 TTL이 하드코딩이다.** `const accessTokenExp = 3600`으로 1시간 고정인데, 실제 access token 만료 시간에서 계산해야 맞다.
3. **env 검증이 없다.** 필수 환경변수가 누락돼도 앱이 그냥 뜬다. zod나 joi로 "필수값 없으면 즉시 실패"를 시키는 게 훨씬 안전하다.
4. **e2e 테스트가 없다.** `package.json`에 스크립트는 있는데 실제 테스트 파일이 없다. 보일러플레이트라면 최소 인증 흐름 e2e 1개는 포함시켜야 한다.
5. **`setGlobalPrefix()`를 안 쓴다.** env에 `API_PREFIX`가 있는데 `main.ts`에서 적용하지 않았다.

---

## 9. Spring Boot 개발자가 느낀 NestJS

Java/Spring Boot를 주력으로 쓰는 입장에서 NestJS를 찍먹하며 느낀 점을 솔직하게 정리한다.

### 닮은 점이 많아서 진입장벽이 낮다

| 개념 | Spring Boot | NestJS |
|------|------------|--------|
| DI 컨테이너 | Spring IoC | NestJS Module + `@Injectable()` |
| 컨트롤러 | `@RestController` | `@Controller()` |
| 미들웨어/필터 | `Filter`, `Interceptor` | `Middleware`, `Guard`, `Interceptor`, `Pipe` |
| 예외 처리 | `@ControllerAdvice` | `@Catch()` ExceptionFilter |
| 설정 관리 | `application.yml` | `ConfigModule` + `.env` |
| API 문서 | SpringDoc/Swagger | `@nestjs/swagger` |
| 스케줄링 | `@Scheduled` | `@Cron()` |
| ORM | JPA/Hibernate | TypeORM |

모듈 단위로 DI를 구성하고, 데코레이터로 횡단 관심사를 처리하는 구조가 Spring과 거의 같다. Spring 경험이 있으면 NestJS 공식 문서를 읽을 때 "아, 그거구나" 하는 순간이 많다.

### NestJS가 좋았던 점

**1. 세팅부터 실행까지가 빠르다.** `nest new`로 프로젝트를 만들면 바로 돌아간다. Spring Boot도 Spring Initializr가 있지만, Gradle 빌드 시간 + JVM 기동 시간을 합치면 체감 차이가 크다. NestJS는 `npm run start:dev`로 핫리로드까지 1~2초면 된다.

**2. 데코레이터 조합이 유연하다.** Spring의 커스텀 어노테이션도 가능하지만 AOP + 리플렉션 조합이 무겁다. NestJS의 `applyDecorators()`는 여러 데코레이터를 함수 하나로 합치는 게 훨씬 간결하다. Swagger 문서화를 커스텀 데코레이터로 묶은 것도 이 덕분이다.

**3. TypeScript의 타입 시스템.** Java의 제네릭보다 유연하다. `ResponseEntity<T>`, DTO 변환, 유니온 타입 같은 걸 적은 코드로 표현할 수 있다.

### Spring Boot가 그리웠던 점

**1. 타입 안전성의 깊이가 다르다.** TypeScript는 결국 런타임에서 타입이 사라진다. Spring Boot + Java는 컴파일 타임에 잡히는 에러가 훨씬 많다. TypeORM 엔티티에서 컬럼 타입을 잘못 매핑해도 빌드가 통과하는 걸 보면서, JPA의 엄격함이 그리웠다.

**2. 트랜잭션 관리가 아쉽다.** Spring의 `@Transactional`은 선언만 하면 AOP가 알아서 처리한다. TypeORM에서 트랜잭션을 쓰려면 `QueryRunner`를 직접 만들거나 `DataSource.transaction()` 콜백을 써야 한다. 서비스 레이어에 트랜잭션 보일러플레이트가 섞이는 느낌이 있다.

**3. 생태계 성숙도 차이.** Spring Security 하나면 OAuth2, SAML, LDAP, 메서드 시큐리티까지 다 커버된다. NestJS에서는 Passport + @nestjs/jwt + Redis + Guard를 직접 조합해야 한다. 이 보일러플레이트의 auth 모듈이 길어진 이유이기도 하다.

**4. 운영 도구.** Spring Boot Actuator, Micrometer, 프로파일링 도구 같은 운영 생태계가 NestJS에는 부족하다. 모니터링을 붙이려면 직접 구성해야 할 게 많다.

### 결론: 찍먹으로 충분했다

NestJS는 Spring Boot 개발자가 Node.js를 써야 할 때 가장 자연스러운 선택이라고 느꼈다. 구조와 철학이 비슷해서 학습 비용이 낮고, 작은 프로젝트나 프로토타입을 빠르게 띄우기에 좋다.

하지만 트랜잭션, 타입 안전성, 운영 도구 측면에서 Spring Boot의 성숙도를 대체하기는 어렵다. 메인 스택을 바꿀 이유까지는 아니었고, "Node.js 환경에서 빠르게 API를 만들어야 할 때 꺼내 쓸 수 있는 뼈대"로 남겨두기로 했다.

---

## 10. 보일러플레이트의 핵심

기능이 많은 게 아니라 **뺄 때 쉽게 빠지는 것**이 핵심이다. 모듈 하나 지웠을 때 다른 데서 에러가 나면 보일러플레이트로 실패한 거다. 이 프로젝트에서 그 기준은 어느 정도 지켰다고 본다. 다음에 손볼 일이 있다면 env 검증과 e2e 샘플을 먼저 추가할 것이다.
