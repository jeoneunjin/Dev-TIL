# [Next.js] Mock 데이터 → Prisma DB 연동 진행 기록

> **Date:** 2026-07-29
>
> **Tag:** #Frontend #Backend #Nextjs #Prisma #Neon #Database #TIL
 
---
 
## 오늘 흐름 한눈에 보기
 
```
캐싱 전략을 실제로 검증하려면 실 DB가 필요하다
    ↓
Neon 프로젝트 생성 (region, auth 설정)
    ↓
Prisma 연결 준비 → init 에러 두 개
    ↓
Prisma 7 스키마 구조 자체가 바뀌어 있었다
    ↓
.env.local 미인식 → 마이그레이션이 로컬로 붙던 문제
    ↓
마이그레이션 성공
    ↓
@prisma/client 경로 문제
    ↓
mock → 실제 쿼리로 옮기며 버그 2개 발견 (404, 500)
    ↓
room.ts 타입을 GetPayload로 정리하며 마무리
```
 
---
 
## 1. 왜 이 작업을 시작했는가
 
이전에 기획을 바꾸면서 서비스 데이터를 공통 데이터와 개인화 데이터로 나누고, 각각 다른 캐싱 전략(revalidate, no-store 등)을 적용하기로 했다. 그런데 이 전략은 mock 데이터로는 검증할 수 없다. 캐싱이든 무효화든 결국 실제 요청이 DB까지 왕복해야 의미가 생기는 개념이기 때문이다. 그래서 mock 데이터를 실제 DB 조회로 바꾸는 작업을 먼저 하기로 했다.
 
---
 
## 2. Neon 프로젝트를 만들기
 
- Project name, Postgres 버전(18)은 기본값 유지
- Region은 아시아(싱가포르)로 변경 — 기본값이 미국 리전이라 한국에서 접속 시 지연이 컸음
- Neon Auth는 꺼둔 상태 유지 — 인증은 Auth.js로 이미 정해서, DB 자체 인증 기능과 충돌하지 않도록
---
 
## 3. Prisma 연결 준비
 
Neon이 발급한 connection string을 `.env.local`에 `DATABASE_URL`로 저장했다. Prisma는 앱 쿼리용 pooled connection과 마이그레이션용 direct connection을 구분해서 쓰는 걸 권장해서 `DIRECT_URL`도 따로 준비했다.
 
---
 
## 4. `prisma init` 한 후 발생한 에러와 해결
 
| 에러 | 원인 | 해결 |
|---|---|---|
| `Cannot find module 'dotenv/config'` | Prisma 7부터 자동 생성되는 `prisma.config.ts`가 dotenv 패키지를 전제로 하지만 자동 설치는 안 해줌 | `npm install dotenv` |
| `Cannot find module 'prisma/config'` | `prisma` CLI가 로컬에 정식 설치되지 않고 `npx`로 임시 실행되고 있었음 | `npm install prisma --save-dev` |
 
---
 
## 5. Prisma 7 스키마 구조
 
Prisma 7은 커넥션을 다루는 방식을 통째로 바꿨다.
 
- `schema.prisma`의 `datasource` 블록에 `url`, `directUrl`을 직접 쓰는 방식이 막혔다. 연결 정보는 `prisma.config.ts`의 `datasource.url`로 옮겨야 한다.
- `PrismaClient` 생성 시 driver adapter(`@prisma/adapter-pg` 등)를 필수로 넘겨야 한다. `new PrismaClient()` 단독 생성은 더 이상 동작하지 않는다.
```ts
// Prisma 7 방식
import { PrismaClient } from "@/lib/generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";
 
const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
export const prisma = new PrismaClient({ adapter });
```
 
기존에는 연결 문자열만 넘기면 끝이었는데, Prisma가 내장 Rust 쿼리 엔진을 빼고 클라이언트를 가볍게 만드는 방향으로 가면서 한 단계 더 명시적인 방식으로 바뀐 것이었다. 이 변화를 모르고 예전 방식대로 접근하다가 문제가 발생하였다.
 
---
 
## 6. `.env.local`이 안 읽혀서 마이그레이션이 로컬로 붙던 문제
 
`prisma.config.ts`의 `import "dotenv/config"`는 `.env`만 자동으로 읽고 `.env.local`은 읽지 않는다. Connection string을 `.env.local`에만 넣어뒀던 게 원인이 되어, 마이그레이션이 실제 Neon 서버 대신 `localhost`로 접속을 시도하다 실패했다(P1001).
 
Next.js는 `.env.local`을 특별 취급하지만 `dotenv` 라이브러리 자체는 `.env`만 기본으로 읽는다는 걸 이번에 알았다. 서로 다른 도구가 서로 다른 파일을 보고 있었던 셈이다. `.env.local` 내용을 `.env`에도 복사해서 해결했다.
 
---
 
## 7. 마이그레이션 성공
 
`npx prisma migrate dev --name init` 실행 → Neon 서버(`ep-purple-lake-...aws.neon.tech`)에 정상 접속, 스키마 반영 완료.
 
---
 
## 8. `@prisma/client`를 못 찾는 문제
 
`Cannot find module '@prisma/client'` 에러가 났다. 원인은 `schema.prisma`의 `generator client` 블록에 커스텀 output 경로(`output = "../lib/generated/prisma"`)가 지정되어 있어서, Prisma Client가 `@prisma/client` 패키지가 아니라 프로젝트 내부 `lib/generated/prisma` 폴더에 직접 생성되는 구조였기 때문이다.
 
- `import { PrismaClient } from "@prisma/client"` → `import { PrismaClient } from "@/lib/generated/prisma/client"`로 경로 수정
- 배포 시 이 생성 폴더가 항상 다시 만들어지도록 `package.json`에 `"postinstall": "prisma generate"` 추가
---
 
## 9. mock → 실제 쿼리로 옮기다가 버그 1 발생 (404)
 
방 목록 페이지(`rooms/page.tsx`)가 여전히 하드코딩된 mock 데이터를 보여주고 있었다.
 
```
mock 데이터의 ID: "okinawa-2026"          ← 카드 클릭 시 이 ID로 이동
DB에 저장된 실제 ID: "cms5ss3o80004pxcjr5kbhdaa"  ← DB에는 이게 있음
```
 
사용자가 방 카드를 클릭하면 `/rooms/okinawa-2026`으로 이동하지만 DB에는 그 ID가 없으니 `notFound()`가 호출되며 404가 났다.
 
```ts
// 이전 — mock 데이터 사용
import { mockRooms } from "@/lib/mock-data";
export default function RoomsPage() {
  // mockRooms.map(...)
}
 
// 수정 후 — DB에서 조회
import { getRooms } from "@/lib/db/room";
export default async function RoomsPage() {
  const rooms = await getRooms(); // 실제 CUID로 링크 생성
}
```
 
목록 페이지 하나만 mock에 남아 있어도 상세 페이지와 ID 체계가 어긋난다는 걸 확인했다.
 
---
 
## 10. 이어서 버그 2 (500) — Prisma와 webpack 충돌
 
404를 고치고 실제 ID로 들어가니 이번엔 방 상세 페이지에서 500이 났다. 원인은 **Prisma 7과 Next.js 13 webpack의 충돌**이었다.
 
Prisma가 자동 생성한 `client.ts`에는 이런 코드가 있다.
 
```ts
import * as path from 'node:path'
import { fileURLToPath } from 'node:url'
globalThis['__dirname'] = path.dirname(fileURLToPath(import.meta.url))
//                                                   ↑ ESM 전용 문법
 
import * as runtime from "@prisma/client/runtime/client"
//                         ↑ 내부적으로는 CJS(CommonJS) 포맷
```
 
Next.js는 서버 컴포넌트를 실행하기 전에 내 코드와 외부 라이브러리를 webpack으로 묶어 하나의 번들로 만든다. 그런데 이 파일 안에는 ESM 문법(`import.meta.url`)과 CommonJS 코드, `node:path` 같은 Node.js 전용 모듈이 섞여 있었다. Node.js는 이 조합을 그대로 이해하지만 webpack은 하나의 번들로 합치는 과정에서 이걸 제대로 처리하지 못했고, 결국 해당 모듈이 webpack 레지스트리에 `undefined`로 등록되면서 `__webpack_modules__[moduleId] is not a function` 에러가 났다.
 
해결은 `next.config.js`에서 webpack이 Prisma 관련 패키지를 아예 번들링하지 않고 Node.js가 그대로 `require()`하도록 지시하는 것이었다.
 
```js
// next.config.js
experimental: {
  serverComponentsExternalPackages: [
    "@prisma/client",      // Prisma 클라이언트 런타임
    "@prisma/adapter-pg",  // PostgreSQL 드라이버 어댑터
    "pg",                  // Node.js pg 패키지
  ],
},
```
 
```
[변경 전] Next.js → webpack → Prisma도 함께 번들링 시도 → 실패
[변경 후] Next.js → webpack(Prisma는 제외) → Node.js가 @prisma/client를 직접 require → 성공
```
 
404는 데이터 소스가 mock에 남아 있어 생긴 애플리케이션 로직 문제였고, 500은 Prisma 내부 구조와 번들러 사이의 런타임 문제였다.
 
---
 
## 11. 마지막`room.ts` 타입을 정리
 
mock 데이터 함수를 실제 Prisma 쿼리로 옮기면서, DB 응답 타입을 손으로 인터페이스로 다시 쓰는 대신 Prisma가 제공하는 `Prisma.RoomGetPayload` 유틸리티 타입을 쓰도록 정리했다.
 
```ts
type RoomWithDetails = Prisma.RoomGetPayload<{
  include: typeof roomInclude;
}>;
```
 
인터페이스를 손으로 쓰면 스키마가 바뀔 때마다(예: `Trip`에 `startDate` 필드 추가) 타입도 같이 고쳐야 하고, 깜빡하면 DB·Prisma·인터페이스가 서로 어긋난다. `GetPayload`를 쓰면 `include`에 지정한 관계까지 포함된 타입이 스키마 변경에 맞춰 자동으로 계산된다.
 
쿼리에 쓰는 `include` 절과 타입 계산에 쓰는 `include` 절이 따로 놀지 않도록 `roomInclude`를 상수로 분리해 하나의 소스로 통일했다. 정산 계산(`computeSettlement`), 총 지출(`totalExpenses`) 같은 순수 함수는 데이터 출처가 mock이든 실제 DB든 영향받지 않아 그대로 재사용했다.
 
---
 
## 오늘 정리한 키워드
 
| 키워드 | 정리 |
|---|---|
| ORM | Prisma처럼 DB 테이블을 코드의 객체·타입으로 다루게 해주는 도구 |
| Migration vs Generate | `migrate`는 DB 스키마 자체를 바꾸는 것, `generate`는 그 스키마에 맞는 타입·클라이언트 코드를 만드는 것 |
| Shadow Database | `migrate dev`가 스키마 변경의 안전성을 미리 검증하기 위해 쓰는 임시 DB |
| Driver Adapter | Prisma 7부터 DB 드라이버(`@prisma/adapter-pg` 등)를 명시적으로 만들어 `PrismaClient`에 주입하는 방식 |
| Singleton 패턴 (`globalForPrisma`) | Next.js 개발 서버의 hot-reload마다 `PrismaClient`가 새로 생성되어 DB 커넥션이 누적되는 걸 막는 패턴 |
| `.env` vs `.env.local` | Next.js는 `.env.local`을 특별 취급하지만, `dotenv`는 기본적으로 `.env`만 읽음 |
| `GetPayload` 유틸리티 타입 | `include`·`select`로 가져온 데이터의 타입을 스키마 변경에 자동으로 맞춰 계산 |
| `postinstall` 스크립트 | `npm install`이 끝난 뒤 자동 실행되는 훅. 배포 환경에서도 `prisma generate`가 빠짐없이 돌게 함 |
| `serverComponentsExternalPackages` | 지정한 패키지를 webpack 번들링 대상에서 제외하고 Node.js가 직접 로드하도록 지시하는 설정 |
 
---
 
## 회고
 
Prisma 7로의 전환 시점과 mock → DB 교체 작업이 겹치면서 설정(connection, adapter) / 런타임(webpack 번들링) / 타입(GetPayload) 세 층위에서 각각 다른 문제를 만났다. 다음에 비슷한 마이그레이션을 할 때는 라이브러리 메이저 버전이 바뀌었는지부터 확인하고 시작하면 시행착오를 줄일 수 있을 것 같다.
 
