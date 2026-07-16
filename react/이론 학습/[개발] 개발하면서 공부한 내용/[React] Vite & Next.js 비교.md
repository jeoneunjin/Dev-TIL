# [React] React(Vite) vs Next.js

> **Date:** 2026-07-16  
> **Tag:** #React #Nextjs #Vite #Frontend #SSR #CSR #SSG #ISR

---

## React와 Next.js의 관계

Next.js는 React 위에서 동작하는 프레임워크이다.

React를 사용하는 방법은 크게 두 가지로 생각하면 된다.

```text
React
   │
   ├── Vite
   │      (SPA 중심)
   │
   └── Next.js
          (SSR/SSG/ISR 지원)
```

- **React** : UI를 만들기 위한 라이브러리
- **Vite** : React 개발을 위한 빌드 도구
- **Next.js** : React 기반의 웹 애플리케이션 프레임워크

따라서 Next.js를 사용한다고 해서 React를 사용하지 않는 것이 아니라, **React를 Next.js 환경에서 사용하는 것**이다.

> 아래 내용은 특별한 언급이 없는 한 **Next.js App Router** 기준으로 작성되었다. (Pages Router에는 Server/Client Component 구분 자체가 없다.)

---

# React(Vite)와 Next.js 개발 방식 차이

## React(Vite)

프로젝트 생성

```bash
npm create vite@latest
```

프로젝트 구조

```text
src/
 ├── App.tsx
 ├── main.tsx
 ├── components/
 └── pages/
```

라우팅은 React Router를 직접 설정한다.

```tsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/about" element={<About />} />
  </Routes>
</BrowserRouter>
```

---

## Next.js

프로젝트 생성

```bash
npx create-next-app@latest
```

프로젝트 구조(App Router)

```text
app/
 ├── layout.tsx
 ├── page.tsx
 ├── about/
 │     └── page.tsx
 └── product/
       └── [id]/
             └── page.tsx
```

폴더가 URL이 된다.

```text
app/page.tsx            → /
app/about/page.tsx      → /about
app/product/[id]/page   → /product/1
```

App Router는 파일 기반 라우팅을 제공하므로 React Router를 별도로 설치할 필요가 없다. (설치해서 못 쓰는 건 아니지만, 실무에서 거의 사용하지 않는다.)

---

# layout.tsx와 page.tsx

React에서는 App.tsx에서 Header와 Footer를 직접 관리했다.

```tsx
<>
  <Header />
  <Routes />
  <Footer />
</>
```

Next.js에서는 layout.tsx가 공통 레이아웃 역할을 한다.

```text
layout.tsx

Header

↓

children(page)

↓

Footer
```

page.tsx는 실제 페이지를 의미한다.

---

# Server Component와 Client Component

Next.js **App Router**의 가장 큰 특징.

## React

모든 컴포넌트가 브라우저에서 실행된다.

```text
Browser

↓

React Component
```

---

## Next.js (App Router)

기본적으로 Server Component이다.

```tsx
export default async function ProductPage() {
    const products = await fetch(...)

    return (...)
}
```

위 코드는 서버에서 실행된다.

---

브라우저에서 실행되어야 하는 경우

```tsx
"use client";
```

를 선언한다.

대표적인 경우

- useState
- useEffect
- 클릭 이벤트
- 입력 이벤트

등 React Hook을 사용하는 컴포넌트

---

정리

| Server Component | Client Component |
|-----------------|------------------|
| 서버에서 실행 | 브라우저에서 실행 |
| JS 번들 감소 | 인터랙션 가능 |
| SEO 유리 | useState 가능 |
| fetch 가능 | 이벤트 처리 |

---

# Next.js의 fetch와 캐싱

React에서는

```tsx
useEffect(() => {
    fetch(...)
}, [])
```

브라우저가 API를 호출한다.

---

Next.js에서는

```tsx
const products = await fetch(...)
```

서버가 데이터를 가져온다.

> ⚠️ **버전별로 기본 캐싱 동작이 다르다.**
> - **Next.js 14 이하**: `fetch()`는 기본값이 `force-cache` → 별도 설정 없이도 자동으로 캐싱됨
> - **Next.js 15 이상**: `fetch()`는 기본값이 `no-store` → 캐싱하려면 명시적으로 옵션을 줘야 함
>
> ```tsx
> // Next.js 15+: 기본은 캐싱 안 함 (매 요청마다 새로 fetch)
> const res = await fetch(url);
>
> // 캐싱하려면 명시적으로 opt-in
> const res = await fetch(url, { cache: "force-cache" });
> const res = await fetch(url, { next: { revalidate: 600 } });
> ```
>
> Vercel 팀이 "암묵적 캐싱이 stale data(오래된 데이터) 버그를 너무 많이 유발한다"는 이유로 15버전부터 이렇게 바꿨다. 프로젝트의 Next.js 버전을 먼저 확인하고 캐싱 전략을 짜야 한다.

---

## React Query 캐싱

React Query는 브라우저(Client)에 캐시한다.

```text
사용자 A

↓

브라우저 캐시

↓

API
```

다른 사용자는

```text
사용자 B

↓

브라우저 캐시

↓

API
```

사용자마다 캐시가 따로 존재한다.

---

## Next.js 캐싱

Next.js는 서버(Server) 쪽에 캐시할 수 있다. (단, 위에서 언급했듯 15버전부터는 명시적으로 옵션을 줘야 캐싱된다.)

```text
사용자 A
      │
사용자 B
      │
      ▼
 Next Server Cache (Data Cache)
      │
      ▼
      DB
```

캐싱을 활성화하면 모든 사용자가 동일한 캐시를 공유한다.

장점

- DB 조회 감소
- 응답 속도 향상
- 서버 부하 감소

### 참고: Next.js의 캐싱은 사실 4단계로 나뉜다

| 캐시 종류 | 위치 | 설명 |
|---|---|---|
| Request Memoization | 서버, 단일 렌더링 내 | 같은 fetch 요청 중복 호출 제거 |
| Data Cache | 서버 | fetch 데이터 캐시 (위에서 다룬 부분) |
| Full Route Cache | 서버 | 빌드 시 생성된 HTML/RSC 결과 캐시 |
| Router Cache | 클라이언트(브라우저) | 페이지 이동 시 RSC 페이로드 캐시 (15버전부터 동적 페이지는 기본적으로 캐싱 안 됨) |

---

# 캐시 유효기간

캐시는 개발자가 직접 결정한다.

예시

```tsx
await fetch(url, {
  next: {
    revalidate: 600
  }
})
```

600초마다 캐시를 갱신한다.

---

실시간 데이터라면

```tsx
await fetch(url, {
    cache: "no-store"
})
```

항상 최신 데이터를 조회한다. (Next.js 15 이상에서는 옵션을 안 줘도 이게 기본값이다.)

---

필요하면 직접 캐시를 삭제할 수도 있다.

```tsx
revalidateTag("products")
```

또는

```tsx
revalidatePath("/products")
```

다음 요청에서 새로운 데이터를 생성한다.

---

# SSR / SSG / ISR

## CSR(Client Side Rendering)

React의 기본 방식

```text
브라우저

↓

JS 다운로드

↓

React 실행

↓

API 호출

↓

화면 생성
```

---

## SSR(Server Side Rendering)

요청마다 서버가 HTML을 생성한다.

```text
사용자

↓

Server

↓

DB

↓

HTML 생성

↓

응답
```

장점

- SEO
- 빠른 첫 화면

단점

- 서버 부담 증가

---

## SSG(Static Site Generation)

빌드 시 HTML 생성

```text
npm run build

↓

HTML 생성

↓

사용자에게 그대로 제공
```

장점

- 매우 빠름

---

## ISR(Incremental Static Regeneration)

중간 방식

예)

10분마다 HTML 재생성

```text
첫 요청

↓

HTML 생성

↓

10분 동안 캐시

↓

10분 후 재생성
```

쇼핑몰, 블로그에서 많이 사용한다.

---

# Route Handlers(app/api)

React에서는 API 서버를 별도로 만든다.

```text
React

↓

Spring Boot
```

---

Next.js에서는

```text
app/
 └── api/
      └── login/
            └── route.ts
```

만들면
/api/login
API가 생성된다.

예시

```tsx
export async function GET() {
    return Response.json({
        hello: "world"
    })
}
```

Spring Controller와 비슷한 역할이다.

---

# Server Actions

기존 방식

```text
Client

↓

POST 요청

↓

API

↓

DB
```

---

Server Action

```tsx
"use server";

export async function saveUser() {

}
```

```tsx
<form action={saveUser}>
```

API를 별도로 만들지 않고 서버 함수를 직접 실행할 수 있다.

---

# next/image

기존 React

```tsx
<img src="/cat.png" />
```

Next.js

```tsx
<Image
    src="/cat.png"
    width={300}
    height={200}
/>
```

자동 제공

- Lazy Loading
- 이미지 최적화
- WebP 변환
- 적절한 크기 제공

---

# next/font

```tsx
import { Inter } from "next/font/google";
```

자동으로

- 폰트 최적화
- 다운로드 최적화
- FOUT/FOIT 감소

---

# Metadata API

React에서는

```html
<title>

<meta>
```

를 직접 관리한다.

---

Next.js

```tsx
export const metadata = {
    title: "상품 상세",
    description: "..."
}
```

자동으로

- title
- meta
- Open Graph

를 생성한다.

SEO에 유리하다.

---

# React(Vite)를 사용하는 경우

다음과 같은 서비스는 React(Vite)가 적합하다.

- 관리자(Admin)
- ERP
- 대시보드
- 게임
- Canvas 프로젝트
- WebRTC
- 채팅 서비스

공통점

- 로그인 후 사용하는 서비스
- SEO가 필요 없음
- 실시간 상호작용이 중요

> 참고: 이런 서비스들을 Next.js로 못 만드는 건 아니다. 전부 Client Component로 구현 가능하다. 다만 SSR/SEO의 이점이 크지 않아서 Vite가 더 가볍고 실용적인 선택이 되는 것뿐이다.

---

# Next.js를 사용하는 경우

다음과 같은 서비스는 Next.js가 적합하다.

- 쇼핑몰
- 블로그
- 뉴스
- 기업 홈페이지
- 여행 서비스
- 부동산
- 맛집 정보 사이트
- 금융 상품 비교 서비스

공통점

- 검색 유입이 중요
- SEO 필요
- 첫 화면 속도가 중요
- SSR/SSG/ISR 활용 가능

---

# 언제 React(Vite), 언제 Next.js?

핵심 판단 기준은

> **"검색엔진이 이 페이지를 검색해야 하는가?"**

YES

↓

Next.js

NO

↓

React(Vite)

---

예시

| 서비스 | React(Vite) | Next.js |
|---------|:----------:|:--------:|
| 관리자 | ✅ | △ |
| ERP | ✅ | △ |
| 게임 | ✅ | △ |
| WebRTC | ✅ | △ |
| 채팅 | ✅ | △ |
| 쇼핑몰 | △ | ✅ |
| 블로그 | ❌ | ✅ |
| 기업 홈페이지 | ❌ | ✅ |
| 뉴스 | ❌ | ✅ |
| 여행 | ❌ | ✅ |

(△ = 기술적으로 구현은 가능하지만 굳이 필요하지 않거나 이점이 적은 경우)

---

# 핵심 정리

| React(Vite) | Next.js |
|-------------|----------|
| React 기반 SPA | React 기반 Full Stack Framework |
| React Router 사용 | App Router 사용 |
| CSR 중심 | SSR, SSG, ISR, CSR 지원 |
| 브라우저에서 실행 | 기본은 Server Component (App Router 기준) |
| React Query로 클라이언트 캐싱 | 서버 캐싱 지원 (버전에 따라 기본 동작 다름) |
| API 서버 별도 필요 | Route Handler, Server Action 제공 |
| SEO 약함 | SEO 강함 |
| 관리자, 게임, 대시보드 | 쇼핑몰, 블로그, 뉴스, 기업 사이트 |

> **React(Vite)는 사용자와의 상호작용이 중심인 SPA에 적합하고, Next.js는 검색 유입과 서버 렌더링이 중요한 웹 서비스에 적합하다.**
