# [React] 슬금 프론트엔드  API 연동 구현 가이드

> **Date:** 2026-03-17  
> 이 문서는 백엔드 API 스펙에 기반한 프론트엔드 전 레이어의 설계 결정, 구현 방법, 코드 흐름을 설명합니다.

---

## 목차

1. [설계 결정  로그인 응답에 세션 정보 포함 여부](#1-설계-결정)
2. [아키텍처 개요](#2-아키텍처-개요)
3. [환경 설정](#3-환경-설정)
4. [API 계층: apiClient + gameService](#4-api-계층)
5. [타입 시스템 (types/)](#5-타입-시스템)
6. [상태 관리 (stores/)](#6-상태-관리)
7. [게임 흐름 상태 머신](#7-게임-흐름-상태-머신)
8. [컴포넌트 변경 사항](#8-컴포넌트-변경-사항)
9. [페이지 구조 및 플로우](#9-페이지-구조-및-플로우)
10. [Google OAuth 설정](#10-google-oauth-설정)
11. [에러 처리 전략](#11-에러-처리-전략)
12. [주요 설계 패턴](#12-주요-설계-패턴)

---

## 1. 설계 결정

### 로그인 응답에 세션 정보를 포함할지 여부

**문제**: 로그인 후 "이어하기" 버튼을 즉시 보여주려면 활성 세션 여부를 알아야 한다.
두 가지 방법이 가능하다:

| 방법 | 장점 | 단점 |
|------|------|------|
| A. 로그인 후 별도 `GET /sessions/active` 호출 | API 단순 분리 | 로그인세션 조회 **waterfall** 발생 (UX 지연) |
| **B. 로그인 응답에 `hasActiveSession` + `session` 포함**  | 로그인 1번으로 모든 정보 취득 | 로그인 응답이 약간 무거워짐 |

 **B를 채택**: `ApiLoginResponse`에 `hasActiveSession?: boolean`, `session?: ApiSessionInfo | null` 필드 추가.
단, 페이지 새로고침 후 토큰만 복원된 상태에서는 세션 정보가 없으므로, `App.tsx`의 `useEffect`에서 `GET /sessions/active`를 별도 호출한다.

```
[로그인 클릭]
   POST /auth/social-login
        응답에 hasActiveSession + session 포함
            hasActiveSession === true  "이어하기" 버튼 즉시 표시

[새로고침 (accessToken 복원)]
   App.tsx useEffect
        GET /sessions/active
            setSessionInfo(hasActive, session)
```

---

## 2. 아키텍처 개요

```
Google GIS (OAuth)
       
       
LoginModal.tsx
    socialLogin()
  
gameService.ts  apiFetch()  백엔드 API
                          
                     useAuthStore  (isLoggedIn, accessToken, sessionInfo)
                     useGameStore  (phase, subPhase, balance, currentEpisode, ...)
                          
               
                                   
          App.tsx     Home.tsx    GamePlay.tsx
                                    
                        
                                              
                  EpisodePage  QuizPage  WeekCompletePage
                  LearningPage
```

### 레이어별 역할

| 레이어 | 파일 | 역할 |
|--------|------|------|
| HTTP 클라이언트 | `services/apiClient.ts` | 인증 헤더 자동 주입, 공통 응답 언래핑, 에러 변환 |
| API 서비스 | `services/gameService.ts` | 엔드포인트별 타입 지정 함수 |
| 인증 상태 | `stores/useAuthStore.ts` | 로그인/로그아웃, accessToken, sessionInfo |
| 게임 상태 | `stores/useGameStore.ts` | 게임 단계, 잔액, 에피소드 데이터, 액션 |
| 페이지 | `pages/game/*.tsx` | 상태에 따른 화면 렌더링 |
| 공통 컴포넌트 | `components/game/**` | 재사용 UI (대사창, 퀴즈카드 등) |

---

## 3. 환경 설정

### 환경 변수 (.env.local)

```bash
# 구글 소셜 로그인 클라이언트 ID (GIS)
VITE_GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com

# 백엔드 API 베이스 URL (프로덕션)
VITE_API_BASE_URL=https://api.example.com

# 로컬 개발 시: 아래 변수 생략  Vite proxy가 /api  localhost:8080 처리
```

### Vite 개발 프록시 (vite.config.ts)

```ts
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
    },
  },
},
```

> `VITE_API_BASE_URL`을 설정하지 않으면 `apiClient.ts`는 `/api` 경로를 사용하므로
> 로컬 개발 환경에서 Vite 프록시를 통해 백엔드와 통신된다.

---

## 4. API 계층

### 4-1. apiClient.ts

```ts
// 공통 응답 래퍼 언래핑: { status, message, data }  data 반환
export async function apiFetch<T>(path: string, options?: RequestInit): Promise<T>
```

- `useAuthStore`를 **동적 import**로 읽어 circular dependency 방지
- Bearer 토큰 자동 주입
- HTTP 에러 시 `ApiException` throw (code, message, status 포함)
- 202 Accepted 등 `data === null`인 경우도 안전하게 처리

### 4-2. gameService.ts  API 함수 목록

| 함수 | HTTP | 엔드포인트 | 설명 |
|------|------|-----------|------|
| `socialLogin(provider, token)` | POST | `/auth/social-login` | 소셜 로그인 |
| `getActiveSession()` | GET | `/sessions/active` | 세션 상태 재조회 |
| `createSession(nickname, gender)` | POST | `/sessions` | 새 게임 세션 생성 |
| `getEpisode(sessionId, month, week)` | GET | `/sessions/{id}/episodes` | 에피소드 조회 |
| `submitChoice(sessionId, episodeId, choiceId)` | POST | `.../choices` | 선택지 제출 |
| `submitQuizAnswers(sessionId, episodeId, answers[])` | POST | `.../quiz-answers` | 퀴즈 전체 답안 배치 제출 |
| `savePoint(sessionId, month, week)` | POST | `.../save-point` | 주차 완료 저장 |
| `createReport(sessionId)` | POST | `.../reports` | 최종 보고서 생성 |

---

## 5. 타입 시스템

### 5-1. types/auth.ts

```ts
export type SocialProvider = 'GOOGLE' | 'KAKAO' | 'NAVER';

export interface User {
  memberId: string;
  name: string;
  email: string;
  provider: SocialProvider;
}

export interface SessionInfo {
  sessionId: string;
  status: string;
  currentMonth: number;
  currentWeek: number;
  nextMonth: number;
  nextWeek: number;
}

export interface AuthState {
  isLoggedIn: boolean;
  accessToken: string | null;
  user: User | null;
  hasActiveSession: boolean;
  sessionInfo: SessionInfo | null;
}
```

### 5-2. types/game.ts  서버 응답 타입

```
ApiDialogue           대사 한 줄 (speaker, talk, characterKey, emotion, sfxKey)
ApiScene              씬 (backgroundKey, bgmKey, dialogues[])
ApiStoryResult        STORY 에피소드 (scenes[], choices[])
ApiQuizQuestion       퀴즈 문항 (choices[isAnswer], description)
ApiQuizResult         QUIZ 에피소드 (questions[])
ApiSeminarResult      SEMINAR 에피소드 (seminarTitle, content, summary)
ApiEpisodeResponse    위 세 가지의 유니온: { type: 'STORY'|'QUIZ'|'SEMINAR', result }
ApiChoiceResult       선택 결과 (balanceDelta, balanceAfter, resultDialogues[])
ApiQuizAnswerItem     배치 제출 단위 (questionId, choiceId)
ApiQuizAnswerResponse 퀴즈 결과 요약 (correctCount, bonusAmount, balanceAfter)
ApiSavePointResponse  세이브 포인트 결과 (nextMonth, nextWeek, balance)
ApiCreateSessionResponse  세션 생성 결과 (sessionId, prologue[])
ApiLoginResponse      로그인 결과 (accessToken, + hasActiveSession, session?)
```

---

## 6. 상태 관리

### 6-1. useAuthStore

```ts
// 상태
isLoggedIn: boolean
accessToken: string | null
user: User | null
hasActiveSession: boolean
sessionInfo: SessionInfo | null

// 액션
login(user, accessToken, hasActiveSession, sessionInfo)
logout()
setSessionInfo(hasActiveSession, sessionInfo)
```

`persist` 미들웨어로 `localStorage`에 저장  새로고침 후에도 로그인 유지.

### 6-2. useGameStore

```ts
// Phase & SubPhase
phase: 'idle' | 'intro' | 'playing' | 'ended'
subPhase: 'prologue' | 'quiz' | 'quiz-result' | 'episode' | 'choice'
          | 'choice-result' | 'learning' | 'week-complete'

// 게임 데이터
sessionId: string | null
month, week: number
balance: number
prologueDialogues: ApiDialogue[]
currentEpisode: ApiStoryResult | null
choiceResult: ApiChoiceResult | null
pendingBalance: number | null   // 결과 대사 완료 후 반영 예정 잔액
currentQuiz: ApiQuizResult | null
quizSummary: ApiQuizAnswerResponse | null
currentSeminar: ApiSeminarResult | null
nextMonth, nextWeek: number | null  // savePoint 응답에서 받은 다음 이동 위치

// 주요 액션
startNewGame()          POST /sessions  subPhase = 'prologue'
resumeGame()            AuthStore에서 세션 정보 읽어 loadEpisode
completePrologue()      loadEpisode(1, 1)
loadEpisode(m, w)       GET /episodes  subPhase = 'episode'|'quiz'|'learning'
advanceToChoice()       subPhase = 'choice'
submitChoice(id)        POST /choices  subPhase = 'choice-result'
completeChoiceResult()  POST /save-point  subPhase = 'week-complete'
submitAllQuizAnswers()  POST /quiz-answers  subPhase = 'quiz-result'
advanceFromQuizResult() POST /save-point  loadEpisode or ended
completeLearning()      POST /save-point  subPhase = 'week-complete'
goToNext()              loadEpisode(nextMonth, nextWeek) or ended
```

---

## 7. 게임 흐름 상태 머신

```
idle
 [startIntro] intro (name  gender  slides)
                   [startNewGame] prologue
                            
                   [completePrologue]
                            
                      loadEpisode(1,1)
                            
              
                                        
           episode         quiz         learning
                                        
    [advanceToChoice]  [submitAll    [completeLearning]
                      QuizAnswers]       
           choice                    week-complete
                       quiz-result       
    [submitChoice]                   [goToNext]
                 [advanceFromQuizResult]  
       choice-result                     
                           
  [completeChoiceResult]                  
                                         
         week-complete 
              
           [goToNext]
              
    
                       
loadEpisode          nextMonth===null
                       
                 createReport()
                 phase='ended'
     (위 분기 반복)
```

---

## 8. 컴포넌트 변경 사항

### DialogueBox / DialogueViewer

- `text`  `talk` 필드 사용
- `speaker` 값 `ME / MOM / FRIEND / PARTNER / NPC / SYSTEM`  한국어 레이블 매핑

### ChoiceModal

- `choiceId: number`  `choiceId: string`
- `label`  `displayCost` (예: "50만원 대출 선택")
- `onSelect: (choiceId: string) => void`

### QuizCard (대폭 변경)

| 구분 | 이전 | 변경 후 |
|------|------|--------|
| 정답 확인 | API 호출 (`POST /quiz/{id}/answer`) | `choice.isAnswer` 클라이언트 즉시 판별 |
| 제출 시점 | 문제마다 즉시 | 마지막 문제 후 `submitAllQuizAnswers()` 1회 |
| Props | `quiz`, `result`, `isLoading`, `onAnswer(number)` | `question: ApiQuizQuestion`, `onAnswer(choiceId: string)`, `onNext()` |

### GenderStep

- `'male'/'female'`  `'MAN'/'WOMAN'` (서버 enum 일치)

### LoginModal

- Google Identity Services (GIS) `initTokenClient` 방식으로 OAuth 구현
- `VITE_GOOGLE_CLIENT_ID` 환경변수 필요
- 로그인 성공  `socialLogin('GOOGLE', access_token)`  `useAuthStore.login()` 호출

---

## 9. 페이지 구조 및 플로우

### EpisodePage (subPhase: prologue / episode / choice / choice-result)

```
prologue:
  prologueDialogues  DialogueViewer  마지막 줄  completePrologue()

episode:
  scenes[sceneIndex]  DialogueViewer
   마지막 씬 마지막 대사  advanceToChoice()

choice:
  ChoiceModal (currentEpisode.choices)  submitChoice(choiceId)

choice-result:
  choiceResult.resultDialogues  DialogueViewer  completeChoiceResult()
```

### QuizPage (subPhase: quiz / quiz-result)

```
quiz:
  currentQuiz.questions[questionIndex]  QuizCard
   handleAnswer(choiceId)  로컬 answers[] 추가
   handleNext():
     - 마지막 문제 아님  questionIndex++
     - 마지막 문제  submitAllQuizAnswers(answers)

quiz-result:
  quizSummary (correctCount, bonusAmount, balanceAfter)
   advanceFromQuizResult()  savePoint  loadEpisode or ended
```

### WeekCompletePage (subPhase: week-complete)

```
nextMonth/nextWeek에 따라 자동으로 버튼 레이블 결정:
  null        "결과 보고서 확인"
  다음 달     "X월 시작"
  동월 week 0  "X월 퀴즈 시작"
  동월 week 4  "X월 학습 자료 보기"
  그 외       "X월 Y주차 시작"

goToNext() 호출 시:
  nextMonth === null  createReport()  phase = 'ended'
  그 외  loadEpisode(nextMonth, nextWeek)
```

### LearningPage (subPhase: learning)

```
currentSeminar (ApiSeminarResult) 표시
   completeLearning()  savePoint(month, 4)
     week-complete 화면으로 전환
```

---

## 10. Google OAuth 설정

`index.html`에 GIS 스크립트 포함 (이미 적용됨):

```html
<script src="https://accounts.google.com/gsi/client" async defer></script>
```

`LoginModal.tsx`에서 초기화:

```ts
const tokenClient = window.google.accounts.oauth2.initTokenClient({
  client_id: import.meta.env.VITE_GOOGLE_CLIENT_ID,
  scope: 'openid email profile',
  callback: async (response) => {
    const result = await socialLogin('GOOGLE', response.access_token);
    authStore.login(
      { memberId: result.memberId, name: result.name, email: result.email, provider: 'GOOGLE' },
      result.accessToken,
      result.hasActiveSession ?? false,
      result.session ?? null,
    );
  },
});
tokenClient.requestAccessToken();
```

---

## 11. 에러 처리 전략

| 상황 | 처리 방식 |
|------|---------|
| API HTTP 에러 | `ApiException` throw  gameStore의 `error` 상태  화면에 에러 메시지 + 재시도 버튼 |
| 401 (토큰 만료) | `App.tsx useEffect`에서 `logout()` 호출 |
| 게임 세션 없음 | `hasActiveSession === false`  "이어하기" 버튼 미표시 |
| 퀴즈 미로드 | 로딩 스피너 표시 (currentQuiz가 null인 상태) |
| 보고서 생성 실패 | `createReport`는 `try/catch`에서 실패 무시하고 `phase = 'ended'` 처리 |

---

## 12. 주요 설계 패턴

### pendingBalance 패턴

선택 결과 대사(choice-result)를 플레이하는 동안 잔액을 즉시 변경하면 대사 내용과 UI 불일치가 발생한다.

 `submitChoice()` 결과에서 받은 `balanceAfter`를 `pendingBalance`에 임시 저장
 결과 대사를 모두 표시한 후 `completeChoiceResult()` 호출 시점에 `balance = pendingBalance` 반영

```ts
// useGameStore.ts
submitChoice: async (choiceId) => {
  const result = await gameService.submitChoice(sessionId, episodeId, choiceId);
  set({ choiceResult: result, pendingBalance: result.balanceAfter, subPhase: 'choice-result' });
},
completeChoiceResult: async () => {
  if (pendingBalance !== null) {
    set({ balance: pendingBalance, pendingBalance: null });
  }
  // savePoint 호출...
},
```

### 배치 퀴즈 제출 패턴

퀴즈는 즉시 피드백이 필요하지만 한 문제씩 API를 호출하면 네트워크 비용이 크다.

 `choice.isAnswer` 필드를 통해 클라이언트에서 즉시 정답 여부 표시
 모든 문제를 풀고 나서 `submitAllQuizAnswers(answers)` 한 번 호출

```ts
// QuizPage.tsx
const handleAnswer = (choiceId: string) => {
  setAnswers(prev => [...prev, { questionId: currentQuestion.questionId, choiceId }]);
};
const handleNext = async () => {
  if (isLast) await submitAllQuizAnswers(answers);
  else setQuestionIndex(i => i + 1);
};
```

### savePoint 호출 시점

| 주차 완료 상황 | savePoint 호출 위치 |
|--------------|-------------------|
| 에피소드 선택 완료 (choice-result 종료) | `completeChoiceResult()` |
| 퀴즈 전체 완료 (quiz-result 종료) | `advanceFromQuizResult()` |
| 학습 자료 확인 (learning 종료) | `completeLearning()` |

savePoint 응답의 `nextMonth: null`은 게임 종료 신호  `createReport()` 호출 후 `phase = 'ended'`

### 세션 복원 (새로고침)

```
localStorage (persist)  accessToken 복원
   App.tsx useEffect (isLoggedIn 감지)
        getActiveSession()
             setSessionInfo(hasActiveSession, session)
                   Home.tsx에서 "이어하기" 버튼 조건부 표시
```

---

*최종 업데이트: 2025년 API 연동 스프린트 완료*
