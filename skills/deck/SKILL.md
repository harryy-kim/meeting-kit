---
name: deck
description: |
  미팅 분석 결과(analysis.md) + 원본 캡처(notes.md) + 배경(context.md) + 슬라이드 이미지(images/)를
  종합하여 사내 공유용 발표 슬라이드(deck.md)를 Markdown(Marp 호환)으로 생성하는 스킬.
  세미 포멀 톤, 한 슬라이드 한 메시지, 핵심 인용·이미지 embed 포함.
  무거운 작성은 deck-builder 서브에이전트에 위임하여 메인 컨텍스트를 보호한다.

  WHEN: "/deck", "발표자료 만들어줘", "이거 발표용으로 정리해줘", "사내 공유 자료",
        "프레젠테이션", "deck", "슬라이드 만들어줘"
  WHEN NOT: 발표 대본(→ script), 종합 분석 문서(→ meeting-review),
            미팅 중 짧은 요약(→ recap)
triggers:
  - /deck
  - 발표자료
  - 발표 자료
  - 사내 공유 자료
  - 프레젠테이션
  - deck
  - 슬라이드 만들
argument-hint: "[meeting-name 또는 비워두면 가장 최근/활성 미팅]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Task
---

# deck — Presentation Slides Generator

미팅 결과를 사내 공유용 발표 슬라이드로 만드는 스킬.

## 언제 사용하나

- 미팅이 끝나고 동료/팀에게 결과를 공유해야 할 때
- analysis.md만으로는 발표하기 어려워 청중 친화적인 자료가 필요할 때
- 사용자가 "발표자료 만들어줘", "사내 공유용으로 정리해줘"라고 말할 때

## 동작

### 1. 대상 미팅 결정

- 인자가 있으면 (`/deck mitsui-q2`) → `~/meetings/*-{mitsui-q2}/` 매칭
- 인자가 없으면 → `~/meetings/.current` (활성 미팅) 우선
- 그것도 없으면 → `~/meetings/` 에서 가장 최근 수정된 폴더
- 매칭 실패 시 사용자에게 후보 목록 제시 후 중단

### 2. 사전 검증

대상 미팅 폴더에 다음이 있는지 확인:
- **`analysis.md` 필수** — 없으면 안내 후 중단:
  ```
  ❌ analysis.md가 없습니다.
  먼저 /meeting-review {meeting-name}로 종합 분석을 생성해주세요.
  ```
- `notes.md` 필수 (분석이 있는데 notes가 없으면 폴더 손상)
- `context.md`, `images/` 는 선택 (있으면 품질 ↑)

기존 `deck.md`가 이미 있으면 덮어쓰기 전 확인:
```
deck.md가 이미 존재합니다. 새로 작성할까요? (y/n)
```

### 3. deck-builder 에이전트 호출

Task 도구로 `deck-builder` 서브에이전트를 호출. 이유:
- 4개 입력 파일(context, analysis, notes, images) 모두 읽고 종합 → 무거운 작업
- 메인 세션 컨텍스트 보호
- 에이전트는 독립 컨텍스트에서 자유롭게 분석·재구성

호출 시 전달:
```
- 미팅 폴더 절대 경로: {abs_path}
- 출력 파일 경로: {abs_path}/deck.md
- 목표 분량: 기본 10장 (analysis.md 토픽 수에 따라 자동 조정)
- 톤: 사내 공유, 세미 포멀, 존댓말
- 활용 원칙: context/analysis/notes/images 모두 활용. 한 가지만 보고 만들지 말 것.
```

### 4. 결과 보고

에이전트가 deck.md 작성을 마치면 사용자에게 짧게:

```
✅ 발표자료 작성 완료 → {meeting-folder}/deck.md

- 슬라이드: N장 (약 T분)
- 사용한 이미지: I장
- 핵심 인용: Q개
- 결정: D개 / 액션: A개

💡 다음: /script {meeting-name}으로 발표 대본도 생성할 수 있습니다.
```

전체 deck 내용을 인라인으로 다 보여주지 말 것. 사용자가 파일을 직접 열어 확인.

## 사용자가 후속 요청을 하면

- "이 슬라이드 빼줘" / "여기 추가해줘" → deck.md를 직접 Read/Edit. 에이전트 재호출 X
- "대본도 만들어줘" → `/script` 스킬로 위임
- "분량 줄여줘" → 에이전트 재호출하되 인자로 목표 슬라이드 수 전달

## Marp 사용 (선택)

생성된 deck.md는 Marp 호환 형식. 사용자가 Marp로 PDF/HTML 슬라이드로 렌더하려면:
```
marp deck.md --pdf
marp deck.md --html
```
Marp가 없어도 일반 마크다운으로 그대로 읽힘.

## 엣지 케이스

| 상황 | 대처 |
|---|---|
| analysis.md 없음 | "/meeting-review를 먼저 실행해주세요" 안내 후 중단 |
| 활성 미팅도 없고 인자도 없음 | "어떤 미팅의 발표자료를 만들까요?" + 최근 미팅 3개 목록 |
| 인자가 모호 (여러 폴더 매칭) | 후보 목록 제시 |
| 기존 deck.md 존재 | 덮어쓰기 전 확인 |
| context.md 비어 있음 | 그대로 진행하되, "💡 context.md를 채우면 다음 발표자료가 더 정확합니다" 한 줄 안내 |

## 주의

- 에이전트 호출은 무겁다. "그냥 슬라이드 한두 장" 같은 요청이면 메인 세션에서 직접 처리
- 절대 `notes.md`, `context.md`, `analysis.md`를 수정하지 말 것 (read-only)
- `deck.md`만 쓴다
