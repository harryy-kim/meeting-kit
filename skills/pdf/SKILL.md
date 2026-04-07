---
name: pdf
description: |
  미팅 분석 리포트(analysis.md)를 PDF로 변환하는 스킬.
  mermaid 다이어그램을 사전에 PNG로 렌더하여 끼워 넣은 뒤 md-to-pdf로 변환한다.
  슬라이드 이미지(images/slide-*.png)와 mermaid 도식이 모두 포함된 단일 PDF 파일을 생성하여
  공유하기 쉽게 만든다.

  WHEN: "/pdf", "분석 PDF로", "리포트 PDF", "PDF로 변환", "PDF로 만들어줘", "공유용 PDF"
  WHEN NOT: 마크다운 그대로 공유 (zip만 필요), HTML 변환, Notion 업로드
triggers:
  - /pdf
  - PDF로 변환
  - PDF로 만들
  - 분석 PDF
  - 리포트 PDF
  - 공유용 PDF
argument-hint: "[meeting-name (optional)]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
---

# pdf — Analysis Report to PDF

미팅 분석 리포트(`analysis.md`)를 mermaid 다이어그램까지 렌더된 PDF로 변환한다.

## 언제 사용하나

- `meeting-review`로 `analysis.md`가 만들어진 뒤, 다른 사람과 공유해야 할 때
- 슬라이드 이미지 + mermaid 다이어그램이 한 파일에 들어가 있는 자료가 필요할 때
- Slack/이메일에 첨부할 단일 파일이 필요할 때

## 동작

### 1. 대상 미팅 결정

- 인자가 있으면 (`/pdf mitsui-q2`) → `~/meetings/*-{mitsui-q2}/` 매칭
- 인자가 없으면 → `~/meetings/.current` (활성 미팅) 우선
- 그것도 없으면 → `~/meetings/`에서 가장 최근 수정된 폴더
- 매칭 실패 시 후보 목록 제시 후 중단

### 2. 사전 체크

대상 미팅 폴더에 `analysis.md`가 있는지 확인.
- 없으면: `❌ analysis.md가 없습니다. 먼저 /meeting-kit:meeting-review로 분석을 생성하세요.` 안내 후 중단
- 있으면 진행

### 3. mermaid 블록 추출 + 렌더

`analysis.md`에서 ` ```mermaid ... ``` ` 블록을 모두 추출해 PNG로 렌더한다.

```bash
cd {meeting-folder}
mkdir -p mermaid-tmp

# 1) 블록 추출
python3 << 'PY'
import re, pathlib
src = pathlib.Path("analysis.md").read_text()
blocks = re.findall(r"```mermaid\n(.*?)\n```", src, re.DOTALL)
for i, b in enumerate(blocks, 1):
    pathlib.Path(f"mermaid-tmp/diagram-{i}.mmd").write_text(b)
print(f"extracted {len(blocks)} mermaid blocks")
PY

# 2) 각 블록을 PNG로 렌더 (mermaid-cli)
for i in $(seq 1 {N}); do
  npx --yes -p @mermaid-js/mermaid-cli mmdc \
    -i mermaid-tmp/diagram-$i.mmd \
    -o mermaid-tmp/diagram-$i.png \
    -b white -w 1600
done
```

mermaid 블록이 0개면 이 단계는 건너뛴다.

### 4. 마크다운 임시본 생성 (블록 → 이미지 참조 치환)

```bash
python3 << 'PY'
import re, pathlib
src = pathlib.Path("analysis.md").read_text()
counter = [0]
def repl(m):
    counter[0] += 1
    return f"![](mermaid-tmp/diagram-{counter[0]}.png)"
out = re.sub(r"```mermaid\n.*?\n```", repl, src, flags=re.DOTALL)
pathlib.Path("analysis-pdf.md").write_text(out)
print(f"replaced {counter[0]} blocks → analysis-pdf.md")
PY
```

### 5. md-to-pdf 실행

```bash
rtk proxy npx --yes md-to-pdf analysis-pdf.md \
  --launch-options '{"args":["--no-sandbox"]}'
```

> **중요**: 사용자 환경에 RTK(rust token killer) 훅이 설치되어 있으면
> 일반 `npx md-to-pdf` 호출이 RTK와 충돌해 실패할 수 있다.
> 이 경우 `rtk proxy npx ...` 형태로 우회한다.
> RTK가 없는 환경이면 `rtk proxy` 접두사 없이도 동작한다 — 둘 다 시도하라.

### 6. 정리

```bash
mv -f analysis-pdf.pdf analysis.pdf
rm analysis-pdf.md
# mermaid-tmp/는 보존 (다음 변환 시 캐시 가능)
```

### 7. 결과 보고

```
✅ PDF 생성 완료 → {meeting-folder}/analysis.pdf ({size}MB)

- 슬라이드 이미지: {slide-count}장 embed
- mermaid 다이어그램: {N}개 렌더
- 총 페이지: (md-to-pdf 출력에서 추출 가능하면)
```

전체 PDF 내용을 인라인으로 보여주지 말 것.

## 의존성

| 도구 | 설치 여부 확인 |
|---|---|
| `python3` | macOS 기본 |
| `npx` | Node.js 설치 시 자동 |
| `@mermaid-js/mermaid-cli` (mmdc) | npx로 자동 다운로드 (첫 실행 시 puppeteer 포함, 시간 걸림) |
| `md-to-pdf` | npx로 자동 다운로드 |
| `rtk` (선택) | 사용자 환경에 RTK 훅이 있을 때만 필요 |

## 엣지 케이스

| 상황 | 대처 |
|---|---|
| `analysis.md` 없음 | meeting-review 먼저 실행하라고 안내 |
| 활성 미팅도 인자도 없음 | 후보 목록 제시 |
| mermaid 블록 0개 | 추출/렌더 단계 건너뛰고 바로 md-to-pdf 실행 |
| md-to-pdf 첫 실행 (puppeteer 다운로드) | timeout 5분 정도로 설정 |
| 기존 analysis.pdf 있음 | 덮어쓰기 (확인 안 물어봄, 재생성이 흔하므로) |
| RTK 없음 | `rtk proxy` 접두사 없이 재시도 |

## 주의

- `analysis.md` 원본은 절대 수정하지 말 것 — 항상 `analysis-pdf.md` 임시본 사용
- `mermaid-tmp/` 폴더는 보존 (사용자가 직접 mermaid PNG가 필요할 수도 있음)
- 변환 실패 시 임시 파일은 그대로 두고 사용자에게 디버깅 정보 제공
- PDF 크기는 슬라이드 수에 따라 5~20MB 정도 — Slack 업로드 한도(50MB) 안에 들어감
