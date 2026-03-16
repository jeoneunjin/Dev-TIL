# [React] 슬기로운 금융생활 프로젝트 게임 상태 관리 및 화면 구현 설계 문서

> **Date:** 2026-03-16  
> **Tag:** #React #Frontend #GameDev #StateManagement #Zustand #TIL

---

## 목차
1. [전체 구조 개요](#1-전체-구조-개요)
2. [상태 머신 설계](#2-상태-머신-설계)
3. [파일 구조](#3-파일-구조)
4. [타입 설계](#4-타입-설계)
5. [서비스 레이어 설계](#5-서비스-레이어-설계)
6. [Zustand 스토어 설계](#6-zustand-스토어-설계)
7. [컴포넌트 설계](#7-컴포넌트-설계)
8. [게임 흐름 상세](#8-게임-흐름-상세)
9. [API 연동 시 교체 방법](#9-api-연동-시-교체-방법)

---

## 1. 전체 구조 개요

```
UI 컴포넌트
    ↓ 상태 구독
Zustand Store (useGameStore)
    ↓ 비즈니스 로직 + API 호출
Service Layer (gameService.ts)
    ↓ 현재는 Mock, 추후 실제 API로 교체
gameService.mock.ts  →  실제 서버 API
```

**핵심 원칙**: Service Layer를 분리하여 API 스펙 확정 후 `gameService.ts` 내부만 교체하면 UI/상태 관리 코드는 수정 없이 동작한다.

---

## 2. 상태 머신 설계

### 최상위 Phase

```
idle ──startIntro──→ intro ──startNewGame──→ playing ──→ ended
                             ──resumeGame──→
```

| Phase | 설명 |
|---|---|
| `idle` | 홈 화면 |
| `intro` | 닉네임/성별 설정 + 스토리 슬라이드 8장 |
| `playing` | 실제 게임 진행 (월/주차/에피소드) |
| `ended` | 6개월 완주 |

### Playing 단계의 SubPhase (상태 머신)

```
[month 1~6] × [week 0~4]

week=0:
  month=1 → 바로 week=1 episode 시작 (1월은 퀴즈 없음)
  month=2~6 → quiz → (퀴즈 완료) → week=1 episode

week=1~3:
  episode → choice → choice-result
     └── episodeIndex 0→1→2 반복
  3번 에피소드 완료 → week-complete → week=4

week=4:
  learning → (완료) → 다음 month or ended
```

### SubPhase 전환 다이어그램

```
quiz
  └─ (모든 퀴즈 완료) → episode [week=1, episodeIndex=0]

episode
  └─ (대화 다 보기) → choice

choice
  └─ (선택지 선택) → choice-result

choice-result
  └─ episodeIndex < 2 → episode [episodeIndex+1]
  └─ episodeIndex = 2 → week-complete

week-complete
  └─ week < 3 → episode [week+1, episodeIndex=0]
  └─ week = 3 → learning [week=4]

learning
  └─ month < 6 → quiz or episode [month+1, week=0 or 1]
  └─ month = 6 → ended
```

---

## 3. 파일 구조

```
src/
├── types/
│   └── game.ts                 ← 모든 게임 관련 타입 정의
│
├── services/
│   ├── gameService.ts          ← API 인터페이스 + 실제 호출 (추후 채움)
│   └── gameService.mock.ts     ← Mock 구현 + 시나리오 데이터
│
├── stores/
│   └── useGameStore.ts         ← Zustand 상태 + 모든 게임 액션
│
├── components/
│   └── game/
│       ├── common/
│       │   ├── GameLayout.tsx       ← 게임 공통 레이아웃 (헤더: 월/주차 + 저축액)
│       │   ├── MonthWeekBadge.tsx   ← (GameLayout에 포함)
│       │   ├── SavingsBadge.tsx     ← (GameLayout에 포함)
│       │   ├── DialogueBox.tsx      ← 대화 한 줄 말풍선
│       │   ├── DialogueViewer.tsx   ← 대화 목록 + 한 줄씩 넘기기
│       │   └── ChoiceList.tsx       ← 선택지 목록
│       └── quiz/
│           └── QuizCard.tsx         ← 퀴즈 문제 + 보기 + 결과
│
└── pages/
    ├── GamePlay.tsx            ← subPhase 라우터 (playing 단계의 진입점)
    └── game/
        ├── EpisodePage.tsx     ← 1~3주차 에피소드 진행
        ├── QuizPage.tsx        ← 2~6월 0주차 퀴즈
        ├── LearningPage.tsx    ← 4주차 학습 내용
        └── WeekCompletePage.tsx ← 주차 완료 화면
```

---

## 4. 타입 설계

### 핵심 타입

```ts
// 게임 진행 위치
type GameSubPhase =
  | 'quiz'           // 0주차 퀴즈
  | 'episode'        // 1~3주차 대화 표시
  | 'choice'         // 선택지 표시
  | 'choice-result'  // 선택 후 결과 대화
  | 'learning'       // 4주차 학습
  | 'week-complete'  // 주차 완료 화면
  | 'month-complete';

// 대화 한 줄 (에피소드, 선택결과, 내레이터 모두 동일 구조)
type DialogueLine = {
  speaker: string;
  text: string;
  avatarUrl?: string;
};

// 에피소드 (2-1 API 응답)
type EpisodeContent = {
  episodeId: number;
  week: number;
  title: string;
  dialogues: DialogueLine[];
  choices: Choice[];
};

// 선택지
type Choice = {
  choiceId: number;
  label: string;      // 버튼 라벨 (짧은 텍스트)
  description: string; // 선택지 설명
  cost: number;       // 음수 = 지출, 0 = 무지출
};

// 선택 결과 (2-2 API 응답)
type ChoiceResult = {
  choiceId: number;
  dialogues: DialogueLine[];
  cost: number;
};

// 퀴즈 (2-0-1 API 응답)
type Quiz = {
  quizId: number;
  question: string;
  options: string[];
};

// 퀴즈 답변 결과 (2-0-2 API 응답)
type QuizAnswerResult = {
  quizId: number;
  correct: boolean;
  explanation?: string; // 오답 시만 반환
};

// 4주차 학습 (2-3 API 응답)
type LearningContent = {
  month: number;
  title: string;
  sections: { heading: string; body: string }[];
};

// 세션 (이어서하기 API 응답)
type GameSession = {
  sessionId: string;
  month: number;
  week: number;
  episodeIndex: number;
  isWeekCompleted: boolean;
  totalSavings: number;
};
```

---

## 5. 서비스 레이어 설계

### 역할 분리

| 파일 | 역할 |
|---|---|
| `gameService.ts` | 함수 시그니처(인터페이스) 정의. 현재는 mock을 호출. 추후 실제 API 호출로 교체. |
| `gameService.mock.ts` | 시나리오 원고 기반 Mock 데이터. 응답 딜레이 300ms 시뮬레이션. |

### API 함수 목록

```ts
createSession({ nickname, gender })    // 1-1. 새 게임 세션 생성
resumeSession()                         // 1-2. 이어서하기
getQuizList(month)                      // 2-0-1. 퀴즈 목록
postQuizAnswer({ quizId, selectedIndex }) // 2-0-2. 퀴즈 답변
getEpisode({ month, week, episodeIndex }) // 2-1. 에피소드 조회
postChoice({ episodeId, choiceId })     // 2-2. 선택지 제출
getLearningContent(month)               // 2-3. 4주차 학습
completeWeek({ month, week })           // 2-4. 주차 완료 갱신
```

---

## 6. Zustand 스토어 설계

### 상태 구조

```ts
// Intro 관련
phase: GamePhase
introStep: number
characterName: string
gender: Gender | null

// Playing 관련
sessionId: string | null
month: number           // 1~6
week: number            // 0~4
episodeIndex: number    // 0~2
subPhase: GameSubPhase
totalSavings: number    // 누적 저축액 (선택 cost의 합산)

// 현재 화면 데이터
currentEpisode: EpisodeContent | null
choiceResult: ChoiceResult | null
quizList: Quiz[]
currentQuizIndex: number
lastQuizResult: QuizAnswerResult | null
learningContent: LearningContent | null

// UX
isLoading: boolean
error: string | null
```

### 주요 액션 흐름

```
startNewGame()
  → createSession() API
  → phase = 'playing', subPhase = 'episode'
  → loadEpisode()

resumeGame()
  → resumeSession() API
  → 진행 위치 복원
  → isWeekCompleted && week==3 → subPhase = 'learning'
  → week==0 && month>1 → subPhase = 'quiz'
  → 그 외 → subPhase = 'episode'

submitChoice(choiceId)
  → postChoice() API
  → totalSavings += result.cost
  → subPhase = 'choice-result'

advanceAfterChoiceResult()
  → episodeIndex < 2 → 다음 에피소드 로드
  → episodeIndex == 2 → completeWeekAndAdvance()

completeWeekAndAdvance()
  → completeWeek() API
  → week==3 → loadLearning(), week=4
  → week==4 → 다음 달로 이동 또는 ended
  → 그 외 → week+1, subPhase='week-complete'
```

---

## 7. 컴포넌트 설계

### GameLayout
- 모든 게임 화면을 감싸는 공통 레이아웃
- 상단 헤더: 월/주차 배지 + 누적 저축액 표시
- `hideHeader` prop으로 주차 완료 화면 등에서 헤더 숨김

### DialogueBox
- 화자별 자동 스타일 분기
  - `내레이터` → 중앙 기울임꼴 지문 스타일
  - `나` → 오른쪽 파란색 말풍선
  - 그 외 → 왼쪽 반투명 말풍선
- `SPEAKER_COLORS` 맵으로 화자별 이름 색상 관리

### DialogueViewer
- `dialogues` 배열을 한 줄씩 표시 (터치/클릭으로 다음 줄)
- 모든 대화 완료 시 `onComplete` 콜백 호출
- 진행 바(n/total) + 완료 시 "선택하기" 버튼으로 전환
- **재사용**: 에피소드 대화, 선택 결과 대화 모두 이 컴포넌트 사용

### ChoiceList & ChoiceButton
- `cost`가 음수이면 "−xx원 지출" 빨간색, 0이면 "0원" 초록색 표시
- 선택 즉시 `disabled` 처리하여 중복 제출 방지

### QuizCard
- 선택 전/후 상태 관리 (`useState` 내부)
- 정답 → 초록색, 오답 → 빨간색 + 해설 표시
- 마지막 문제에서 "다음 문제" → "게임 시작!" 으로 버튼 텍스트 변경

---

## 8. 게임 흐름 상세

### 1월 흐름 (퀴즈 없음)

```
새 게임 시작
  → createSession()
  → month=1, week=1, episodeIndex=0
  → EpisodePage (1월 1주차 1번 에피소드)
  → 대화 → 선택지 → 선택 결과
  → episodeIndex=1 (2번 에피소드)
  → 대화 → 선택지 → 선택 결과
  → episodeIndex=2 (3번 에피소드)
  → 대화 → 선택지 → 선택 결과
  → WeekCompletePage (1주차 완료)
  → 2주차 시작...
  → 3주차 완료 → LearningPage (4주차 학습)
  → 2월로 이동
```

### 2~6월 흐름 (퀴즈 있음)

```
다음 달(예: 2월) 시작
  → month=2, week=0
  → QuizPage (2월 0주차 퀴즈)
  → 3문제 퀴즈 완료
  → EpisodePage (2월 1주차)
  → ... (1월과 동일한 에피소드 흐름)
```

### 이어서하기 분기

| 저장된 상태 | 재시작 화면 |
|---|---|
| week=2, isWeekCompleted=false | EpisodePage (2주차 episodeIndex부터) |
| week=3, isWeekCompleted=true | LearningPage (4주차 학습) |
| week=0, month>1 | QuizPage |
| week=1, isWeekCompleted=false | EpisodePage |

---

## 9. API 연동 시 교체 방법

### Step 1: `gameService.ts`의 각 함수 내부만 교체

```ts
// Before (mock 호출)
export async function getEpisode(params) {
  return mockGetEpisode(params);
}

// After (실제 API 호출)
export async function getEpisode(params) {
  const res = await fetch(`/api/episodes?month=${params.month}&week=${params.week}&index=${params.episodeIndex}`);
  return res.json();
}
```

### Step 2: 타입 필드명이 다를 경우

`types/game.ts`에서 필드명만 수정하면 TypeScript 컴파일러가 오류 위치를 모두 표시해줌.

### Step 3: `gameService.mock.ts`는 그대로 유지

테스트, 오프라인 개발 용도로 계속 활용 가능.

---

## 구현 현황 체크리스트

- [x] `types/game.ts` — 전체 타입 정의 완료
- [x] `services/gameService.ts` — API 인터페이스 + Mock 연결
- [x] `services/gameService.mock.ts` — 1~2월 시나리오 데이터 완성
- [x] `stores/useGameStore.ts` — 전체 상태 머신 + 액션 구현
- [x] `components/game/common/GameLayout.tsx` — 공통 레이아웃
- [x] `components/game/common/DialogueBox.tsx` — 대화 말풍선
- [x] `components/game/common/DialogueViewer.tsx` — 대화 뷰어
- [x] `components/game/common/ChoiceList.tsx` — 선택지 목록
- [x] `components/game/quiz/QuizCard.tsx` — 퀴즈 카드
- [x] `pages/GamePlay.tsx` — subPhase 라우터
- [x] `pages/game/EpisodePage.tsx` — 에피소드 진행 페이지
- [x] `pages/game/QuizPage.tsx` — 퀴즈 페이지
- [x] `pages/game/LearningPage.tsx` — 4주차 학습 페이지
- [x] `pages/game/WeekCompletePage.tsx` — 주차 완료 화면
- [x] `pages/GameIntro.tsx` — 마지막 슬라이드에서 `startNewGame` 연결
- [x] `App.tsx` — `GamePlay` 컴포넌트 연결 + 이어서하기 버튼 연결

## 남은 작업 (API 확정 후)

- [ ] `gameService.ts` — 실제 API 엔드포인트 연결
- [ ] 3~6월 Mock 에피소드 데이터 추가 (`gameService.mock.ts`)
- [ ] 3~6월 퀴즈 데이터 추가
- [ ] 아바타 이미지 asset 연결 (`DialogueLine.avatarUrl`)
- [ ] 에피소드별 배경 이미지(에셋) 연결
- [ ] 이어서하기 세션 로컬스토리지 → 실제 서버 저장으로 전환
