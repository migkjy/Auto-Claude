# JARVIS 확장 UI 기획서

> 생성일: 2026-02-02
> 기반: Auto-Claude UI (변경 최소화)

---

## 개발 원칙

### 기존 코드 최대 활용
- 신규 컴포넌트 생성 전, 기존 컴포넌트로 해결 가능한지 먼저 검토
- 기존 UI 패턴/스타일 그대로 사용
- shadcn/ui 컴포넌트 재사용

### 변경 최소화
- 신규 페이지 생성 안 함
- 기존 칸반에 필터 추가로 해결
- 기존 Insights를 자비스 채팅으로 활용

---

## 1. 변경 사항 요약

| 영역 | 변경 방법 | 신규 파일 |
|------|----------|----------|
| 비즈니스 태스크 | 기존 칸반 + 카테고리 필터 | 없음 |
| 자비스 채팅 | 별도 UI 생성 (Insights 구조 참고) | `Jarvis.tsx` |
| 카테고리 추가 | 기존 상수 파일 확장 | 없음 |

---

## 2. 칸반 필터 추가

### 현재 Auto-Claude 칸반
```
┌────────────────────────────────────────────────────────────────────┐
│  프로젝트: [전체 ▼]                                    [+ 새 태스크] │
├────────────────────────────────────────────────────────────────────┤
│  backlog | queue | in_progress | ai_review | human_review | done   │
└────────────────────────────────────────────────────────────────────┘
```

### 변경 후 (필터 추가)
```
┌────────────────────────────────────────────────────────────────────┐
│  [Expand All] [전체|개발|비즈니스]           [Refresh] [+ 새 태스크] │
│               ↑ 토글 필터 추가 (좌측 헤더)                          │
├────────────────────────────────────────────────────────────────────┤
│  backlog | queue | in_progress | ai_review | human_review | done   │
└────────────────────────────────────────────────────────────────────┘
```

### 구현 상세 (검증 완료 ✅)

**구현 위치**: `KanbanBoard.tsx`
- 헤더 영역 (line 1523-1554): `<div className="flex items-center gap-2">` 내부
- 필터 로직 (line 719-724): `filteredTasks` useMemo 확장

**필요 상태**:
```typescript
const [categoryFilter, setCategoryFilter] = useState<'all' | 'development' | 'business'>('all');
```

**필터 로직 확장**:
```typescript
const filteredTasks = useMemo(() => {
  let tasks = showArchived ? tasks : tasks.filter(t => !t.metadata?.archivedAt);

  if (categoryFilter === 'development') {
    tasks = tasks.filter(t => isDevelopmentCategory(t.metadata?.category));
  } else if (categoryFilter === 'business') {
    tasks = tasks.filter(t => isBusinessCategory(t.metadata?.category));
  }

  return tasks;
}, [tasks, showArchived, categoryFilter]);
```

**UI 컴포넌트**: 기존 `Button` 컴포넌트 3개로 토글 구현 (shadcn/ui)

---

## 3. 비즈니스 카테고리 배지

### 기존 개발 카테고리 (유지)
| 카테고리 | 배지 | 색상 |
|----------|------|------|
| feature | Feature | Primary |
| bug_fix | Bug Fix | Destructive |
| refactoring | Refactoring | Cyan |
| documentation | Docs | Amber |
| ... | ... | ... |

### 추가할 비즈니스 카테고리
| 카테고리 | 배지 | 색상 |
|----------|------|------|
| research | 📊 Research | Indigo |
| planning | 📋 Planning | Purple |
| marketing | 📢 Marketing | Orange |
| video | 🎬 Video | Red |
| document | 📝 Document | Green |
| automation | 🤖 Automation | Cyan |

**구현 위치**: `shared/constants/task.ts`

---

## 4. 태스크 카드 (변경 없음)

기존 `TaskCard.tsx` 그대로 사용. 카테고리 배지만 새 색상으로 표시됨.

```
┌─────────────────────────────────────┐
│ 📊 Research                         │  ← 새 카테고리 배지
│ ───────────────────────────────────  │
│ 부고장 서비스 시장 규모와           │
│ 경쟁사 분석 수행                    │
│                                     │
│ ████████░░░░░░░░  45%               │
│                                     │
│ 📁 부고 비즈니스    🕐 2h ago       │
└─────────────────────────────────────┘
```

---

## 5. 자비스 채팅 UI (Jarvis.tsx 신규 생성)

### Insights와 분리하는 이유

| 구분 | Insights | Jarvis |
|------|----------|--------|
| 용도 | 코드베이스 탐색/분석 | 비즈니스 프로젝트 관리 |
| 바인딩 | projectId 종속 | 크로스 프로젝트 (독립) |
| 컨텍스트 | 코드/파일 관련 질문 | 비즈니스 태스크 생성/관리 |
| 사이드바 위치 | 기존 위치 유지 | 새 메뉴 항목 추가 |

### Jarvis.tsx 구조 (Insights 참고)

```
┌───────────────────────────────────────────────────────────────┐
│ 🤖 JARVIS                                    [모델: opus ▼]   │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 👤 User                                                 │  │
│  │ "부고장 서비스 비즈니스를 시작하고 싶어"                │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 🤖 JARVIS                                               │  │
│  │                                                         │  │
│  │ 이 비즈니스를 위해 다음 태스크들을 생성할 수 있습니다:  │  │
│  │                                                         │  │
│  │ 1. 📊 시장조사 - 부고장 서비스 시장 분석                │  │
│  │ 2. 📋 비즈니스 기획 - 수익 모델 설계                    │  │
│  │ 3. 💻 웹서비스 개발 - MVP 개발                          │  │
│  │                                                         │  │
│  │ [프로젝트 생성] [태스크만 생성]                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│ [메시지 입력...]                                    [전송]    │
└───────────────────────────────────────────────────────────────┘
```

### 활용할 기존 기능

| 기능 | 위치 | 활용 방법 |
|------|------|----------|
| 채팅 UI 구조 | Insights.tsx | 레이아웃 참고 |
| 메시지 상태 관리 | insights-store.ts | 패턴 참고하여 jarvis용 구현 |
| 태스크 생성 | `createTaskFromSuggestion` | 그대로 import 사용 |
| 스트리밍 응답 | sendMessage 구현 | 패턴 참고 |

**신규 파일**: `components/Jarvis.tsx` (Insights 구조 참고)

---

## 6. 상태 변경 리스너 확장

### 기존 로직 (KanbanBoard.tsx)
```typescript
// 태스크가 in_progress 떠나면 → queue에서 다음 태스크
if (oldStatus === 'in_progress' && newStatus !== 'in_progress') {
  processQueue();
}
```

### 확장 로직 (추가)
```typescript
// 비즈니스 태스크 완료 시 → 다음 행동 결정
if (newStatus === 'done' && isBusinessCategory(task.category)) {
  // 1. 프로젝트에 다음 태스크 있으면 → 자동 시작
  // 2. 없으면 → 알림만
}

// 사용자 승인 필요 시
if (newStatus === 'human_review') {
  // 알림 표시 (기존 notification 시스템)
}
```

---

## 7. 구현 순서

### Phase 1: 카테고리 추가 (30분)
1. `shared/types/task.ts` - TaskCategory 타입 확장
2. `shared/constants/task.ts` - 라벨, 색상 추가

### Phase 2: 필터 UI (1시간)
1. `KanbanBoard.tsx` - 카테고리 필터 토글 추가

### Phase 3: 자비스 로직
1. `KanbanBoard.tsx` - listener 확장
2. (선택) `hooks/useJarvisListener.ts` - 로직 분리

### Phase 4: 자비스 채팅 UI
1. `components/Jarvis.tsx` - 비즈니스 AI 채팅 (Insights 구조 참고)

---

## 8. 생성하지 않는 것

| 항목 | 이유 |
|------|------|
| ~~BusinessKanban.tsx~~ | 기존 칸반 + 필터 |
| ~~jarvis-store.ts~~ | 기존 insights-store 패턴 활용 |
| ~~별도 칸반 페이지~~ | 기존 칸반 + 필터로 해결 |

### 신규 생성할 것

| 항목 | 이유 |
|------|------|
| `Jarvis.tsx` | Insights와 용도 분리 (코드탐색 vs 프로젝트관리) |

---

## 변경 이력

| 날짜 | 변경 내용 |
|------|-----------|
| 2026-02-02 | 최초 작성 (Auto-Claude 기반 신규 UI) |
| 2026-02-02 | 기존 시스템 활용으로 대폭 단순화 |
| 2026-02-02 | Jarvis.tsx 별도 생성으로 변경 (Insights와 분리) |
| 2026-02-02 | 칸반 필터 토글 구현 상세 추가 (검증 완료) |
