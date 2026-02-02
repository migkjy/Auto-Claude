# JARVIS 멀티 에이전시 시스템 - Auto-Claude 기반

> 수정일: 2026-02-02
> 기반: Auto-Claude v2.7.6-beta.2

---

## 개발 원칙 (필수)

### 1. 기존 코드 최대 활용
- **신규 파일 생성 전**, 기존 시스템으로 해결 가능한지 먼저 검토
- **기존 컴포넌트 복사보다** 확장/수정 우선
- **기존 store 활용** - 새 store 생성은 최후의 수단

### 2. 변경 최소화
- 한 줄 수정으로 해결 가능하면, 새 함수 만들지 않음
- 기존 타입 확장 > 새 타입 정의
- 기존 상수에 값 추가 > 새 상수 파일 생성

### 3. Auto-Claude 플로우 존중
- 기존 태스크 실행 파이프라인 그대로 사용
- 기존 상태 관리 패턴 유지
- 기존 IPC 채널 활용

---

## 프로젝트 목표

**개발 외 다양한 비즈니스 작업을 자율 에이전시로 자동화**

- 마케팅 기획 및 실행
- 비즈니스 기획
- 영상 기획 / 유튜브 스크립트
- 판매 자동화
- 기타 반복적/복잡한 비즈니스 태스크

---

## 핵심 발견: Auto-Claude 기존 시스템

### 이미 있는 기능들

| 필요한 기능 | Auto-Claude에 이미 있음 | 위치 |
|-------------|------------------------|------|
| 태스크 완료 감지 | `registerTaskStatusChangeListener` | task-store.ts |
| 다음 태스크 자동 실행 | `processQueue()` | KanbanBoard.tsx |
| AI 채팅 + 태스크 생성 | Insights 페이지 | Insights.tsx |
| 에이전트 프로필 | Agent Profiles | claude-profile-store.ts |
| 알림 시스템 | NotificationSettings | project.ts |
| 프로젝트 관리 | Project 시스템 | project-store.ts |

### 핵심 메커니즘: Task Status Change Listener

```typescript
// KanbanBoard.tsx (line 1206-1221)
registerTaskStatusChangeListener((taskId, oldStatus, newStatus) => {
  // 태스크가 in_progress를 떠나면 → queue에서 다음 태스크 자동 시작
  if (oldStatus === 'in_progress' && newStatus !== 'in_progress') {
    processQueue();
  }
});
```

**이 리스너를 확장하면 자비스 기능 구현 가능!**

---

## MVP 구현 계획

### Phase 1: 비즈니스 카테고리 추가 (변경 2개 파일)

**목표**: 기존 칸반에서 비즈니스 태스크 관리

**변경 파일:**
1. `shared/types/task.ts` - TaskCategory 타입 확장
2. `shared/constants/task.ts` - 비즈니스 카테고리 추가

```typescript
// task.ts - 기존 카테고리에 추가
export type TaskCategory =
  | 'feature' | 'bug_fix' | 'refactoring' | ... // 기존
  | 'research'      // 📊 리서치
  | 'planning'      // 📋 기획
  | 'marketing'     // 📢 마케팅
  | 'video'         // 🎬 영상
  | 'document'      // 📝 문서
  | 'automation';   // 🤖 자동화
```

---

### Phase 2: 자비스 로직 추가 (확장 1개 파일)

**목표**: 태스크 완료 시 자동으로 다음 행동 결정

**변경 파일:**
1. `KanbanBoard.tsx` - 기존 listener 확장

```typescript
registerTaskStatusChangeListener((taskId, oldStatus, newStatus) => {
  // 기존 로직 유지
  if (oldStatus === 'in_progress' && newStatus !== 'in_progress') {
    processQueue();
  }

  // 자비스 확장: 비즈니스 태스크 완료 시 다음 행동
  if (newStatus === 'done') {
    const task = getTaskById(taskId);
    if (isBusinessCategory(task.metadata?.category)) {
      // Insights AI로 결과 분석 → 다음 태스크 생성
      analyzeAndCreateNextTask(task);
    }
  }
});
```

**또는** 별도 hook으로 분리:
1. `hooks/useJarvisListener.ts` - 자비스 로직만 분리 (선택)

---

### Phase 3: 칸반 카테고리 필터 추가 (수정 1개 파일)

**목표**: 개발/비즈니스 태스크 분리 보기

**변경 파일:**
1. `KanbanBoard.tsx` - 헤더에 필터 토글 추가

```
[ 전체 | 개발 | 비즈니스 ]  ← 토글로 칸반 필터링
```

**구현 위치 검증 완료:**
- 헤더 영역 (line 1523-1554): 좌측 "Expand All" 옆에 토글 추가
- 필터 로직 (line 719-724): `filteredTasks` useMemo에 카테고리 필터 체이닝
- 충돌 없음 확인됨 ✅

**필요 헬퍼 함수** (`shared/constants/task.ts`에 추가):
```typescript
export const BUSINESS_CATEGORIES = ['research', 'planning', 'marketing', 'video', 'document', 'automation'] as const;
export const DEVELOPMENT_CATEGORIES = ['feature', 'bug_fix', 'refactoring', 'documentation', 'security', 'performance', 'ui_ux', 'infrastructure', 'testing'] as const;

export function isBusinessCategory(category?: string): boolean {
  return BUSINESS_CATEGORIES.includes(category as any);
}
export function isDevelopmentCategory(category?: string): boolean {
  return DEVELOPMENT_CATEGORIES.includes(category as any);
}
```

---

### Phase 4: 자비스 채팅 UI (신규 1개 파일)

**목표**: 비즈니스 프로젝트 관리를 위한 AI 채팅 인터페이스

**신규 파일:**
1. `components/Jarvis.tsx` - Insights 구조 참고하되 별도 용도

**Insights vs Jarvis 분리 이유:**
| 기능 | Insights | Jarvis |
|------|----------|--------|
| 용도 | 코드베이스 탐색/분석 | 프로젝트 관리/자동화 |
| 바인딩 | projectId 종속 | 크로스 프로젝트 |
| 컨텍스트 | 코드/파일 관련 질문 | 비즈니스 태스크 관련 |

**Jarvis 핵심 기능:**
- 사용자와 대화하며 비즈니스 태스크 생성
- 프로젝트 생성/관리 (creatTaskFromSuggestion 활용)
- 태스크 완료 결과 분석 및 다음 행동 제안

---

## 자비스 역할 정의

### 자비스 = Jarvis.tsx + 확장된 Listener

```
태스크 완료
    │
    ▼
Status Change Listener (기존)
    │
    ├── queue 처리 (기존 로직)
    │
    └── 자비스 로직 (확장)
            │
            ▼
        비즈니스 태스크인가?
            │
            ├── Yes → Insights AI로 분석
            │           │
            │           ├── 다음 태스크 자동 생성 (createTaskFromSuggestion)
            │           └── 또는 human_review로 변경 (사용자 승인 필요 시)
            │
            └── No → 기존 로직 유지
```

### 자비스가 하는 일

1. **태스크 완료 감지** - 기존 listener 활용
2. **결과 분석** - Insights AI (sendMessage) 활용
3. **다음 행동 결정**:
   - 연속 작업이면 → 다음 태스크 자동 생성 (queue에 추가)
   - 사용자 확인 필요하면 → human_review로 상태 변경
   - 모든 작업 완료면 → 완료 알림 (notification)

---

## 수정 대상 파일 요약

### 필수 변경 (3개)

| 파일 | 변경 내용 | 라인 수 |
|------|----------|--------|
| `shared/types/task.ts` | TaskCategory 타입에 6개 추가 | ~10줄 |
| `shared/constants/task.ts` | 카테고리 라벨/색상 + 헬퍼 함수 | ~50줄 |
| `KanbanBoard.tsx` | 카테고리 필터 토글 + listener 확장 | ~40줄 |

### 신규 생성 (1개)

| 파일 | 변경 내용 | 참고 |
|------|----------|------|
| `components/Jarvis.tsx` | 비즈니스 AI 채팅 UI | Insights 구조 참고 |

### 선택 변경 (1개)

| 파일 | 변경 내용 | 필요 시 |
|------|----------|--------|
| `hooks/useJarvisListener.ts` | 자비스 로직 분리 | 코드 정리 시 |

### 생성 안 함

- ~~BusinessKanban.tsx~~ → 기존 칸반 + 필터
- ~~jarvis-store.ts~~ → 기존 store 활용 (insights-store 패턴)
- ~~agency-runner.ts~~ → 기존 Agent Profile 활용

---

## 참고: Auto-Claude 관련 파일

### 태스크 시스템
- `stores/task-store.ts` - 태스크 상태 관리, listener 등록
- `components/KanbanBoard.tsx` - 칸반 UI, processQueue, listener 사용
- `main/agent/agent-manager.ts` - 태스크 실행 관리

### AI 채팅
- `components/Insights.tsx` - 채팅 UI
- `stores/insights-store.ts` - 채팅 상태, sendMessage, createTaskFromSuggestion

### 타입/상수
- `shared/types/task.ts` - TaskCategory, TaskStatus 등
- `shared/constants/task.ts` - 카테고리 라벨, 색상

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|-----------|
| 2026-02-02 | Auto-Claude 기반으로 전면 재작성 |
| 2026-02-02 | 기존 시스템 최대 활용 방향으로 단순화 |
| 2026-02-02 | 개발 원칙 (기존 코드 활용, 변경 최소화) 추가 |
| 2026-02-02 | Jarvis.tsx 별도 생성으로 변경 (Insights와 분리) |
| 2026-02-02 | KanbanBoard.tsx 카테고리 필터 토글 검증 완료 |
