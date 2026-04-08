---
name: script
description: |
  생성된 발표 슬라이드(deck.md)를 입력으로 받아 발표자가 그대로 읽을 수 있는
  발표 대본(script.md)을 생성하는 스킬. 사내 공유 톤(존댓말, 세미 포멀).
  슬라이드별 본문 + 강조 포인트 + 전환 멘트 + 예상 질문 + 시간 배분 포함.
  무거운 작성은 script-writer 서브에이전트에 위임하여 메인 컨텍스트를 보호한다.

  WHEN: "/script", "발표 대본", "발표 스크립트", "발표 멘트", "이거 어떻게 말해야 해",
        "발표 연습", "리허설"
  WHEN NOT: 슬라이드 자체 생성(→ deck), 종합 분석(→ meeting-review)
triggers:
  - /script
  - 발표 대본
  - 발표 스크립트
  - 발표 멘트
  - 발표 연습
  - 리허설
argument-hint: "[meeting-name] [--duration N]  (예: /script mitsui-q2 --duration 15)"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Task
---

# script — Presentation Script Generator

발표 슬라이드를 발표자가 그대로 읽을 수 있는 **구어체 대본**으로 변환하는 스킬.

## 언제 사용하나

- `/deck`로 발표자료를 만든 뒤, 실제 발표를 준비할 때
- 슬라이드만으로는 무슨 말을 할지 막막할 때
- 외국어 미팅 결과를 한국어로 말로 전달해야 하는데 표현이 자연스럽지 않을까 걱정될 때

## 동작

### 1. 인자 파싱

```
/script [meeting-name] [--duration N]
```

- `meeting-name` 없으면 활성 미팅 → 가장 최근 미팅 순으로 결정
- `--duration N` (분 단위) — 목표 발표 시간. 없으면 슬라이드 수 × 1분 기본

### 2. 사전 검증

- **`deck.md` 필수** — 없으면 안내 후 중단:
  ```
  ❌ deck.md가 없습니다.
  먼저 /deck {meeting-name}로 발표자료를 생성해주세요.
  ```
- `notes.md`, `context.md`는 보조 (있으면 인용·톤 조정에 활용)

기존 `script.md`가 있으면 덮어쓰기 전 확인:
```
script.md가 이미 존재합니다. 새로 작성할까요? (y/n)
```

### 3. script-writer 에이전트 호출

Task 도구로 `script-writer` 서브에이전트 호출.

호출 시 전달:
```
- 미팅 폴더 절대 경로: {abs_path}
- deck.md 경로: {abs_path}/deck.md
- 출력 파일 경로: {abs_path}/script.md
- 목표 발표 시간: {N분 또는 "기본 (슬라이드당 1분)"}
- 톤: 사내 공유, 세미 포멀, 존댓말, 자연스러운 구어체
- 활용 원칙: deck.md 순서·메시지 절대 준수. notes.md는 인용·디테일 보강에만 활용.
```

### 4. 결과 보고

```
✅ 발표 대본 작성 완료 → {meeting-folder}/script.md

- 슬라이드 대본: N개
- 목표 발표 시간: 약 T분
- 강조 포인트: N개
- 예상 질문: Q개

💡 발표 전 한 번 소리 내어 읽어보시길 권합니다.
```

전체 대본을 인라인으로 보여주지 말 것.

## deck.md ↔ script.md 일관성

deck.md를 수정한 뒤에는 script.md도 다시 생성하는 것이 좋다. 그렇지 않으면 슬라이드와 대본이 어긋남.

사용자가 `/deck`을 다시 돌렸을 때:
- script.md가 이미 존재하면 → "deck이 갱신됐습니다. /script로 대본도 다시 생성하세요" 안내

## 사용자가 후속 요청을 하면

- "이 슬라이드 멘트만 다시 써줘" → script.md를 직접 Read/Edit. 에이전트 재호출 X
- "발표 시간을 줄여줘" → 에이전트 재호출 (`--duration` 새로 지정)
- "더 캐주얼하게" / "더 포멀하게" → 에이전트 재호출하되 톤 인자 추가

## 엣지 케이스

| 상황 | 대처 |
|---|---|
| deck.md 없음 | "/deck를 먼저 실행해주세요" 안내 후 중단 |
| 활성 미팅도 없고 인자도 없음 | "어떤 미팅의 대본을 만들까요?" + 최근 미팅 3개 목록 |
| 기존 script.md 존재 | 덮어쓰기 전 확인 |
| `--duration`이 슬라이드 수보다 너무 짧음 (예: 10장 5분) | 경고 후 진행, 각 슬라이드 30초 미만 |

## 주의

- `deck.md`, `notes.md`, `context.md`, `analysis.md`는 모두 read-only
- `script.md`만 쓴다
- 결과는 한국어 존댓말 구어체
