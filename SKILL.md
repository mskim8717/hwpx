---
name: hwpx
description: "한글(HWPX) 문서 생성/읽기/편집 스킬. .hwpx 파일, 한글 문서, Hancom, OWPML 관련 요청 시 사용."
---

# HWPX 문서 생성 스킬 — XML-first 워크플로우

한글(Hancom Office)의 HWPX 파일을 **XML 직접 작성** 중심으로 생성, 편집, 읽기할 수 있는 스킬.
HWPX는 ZIP 기반 XML 컨테이너(OWPML 표준)이다. python-hwpx API의 서식 버그를 완전히 우회하며, 세밀한 서식 제어가 가능하다.
**지원 템플릿**: `base`(빈 문서 스켈레톤), `report`(보고서 양식)

## 두 가지 모드 구분 (필수)

| | 레퍼런스 복원 모드 | 템플릿 기반 생성 모드 |
|---|---|---|
| **언제** | 사용자가 .hwpx 파일을 첨부했을 때 | 첨부 파일 없이 새 문서 요청 시 |
| **header.xml** | 레퍼런스에서 추출, 그대로 사용 | 템플릿 것 그대로 사용 |
| **section0.xml** | 레퍼런스 구조를 **최대한 보존**, 텍스트만 치환 | secPr만 복사, **본문은 내용에 맞게 새로 작성** |
| **문단/표 수** | 레퍼런스와 동일하게 유지 | 내용 분량에 맞게 자유롭게 결정 |
| **쪽수** | 레퍼런스와 100% 동일 필수 | 제한 없음 |

## 모드 1: 레퍼런스 복원 (사용자가 .hwpx를 첨부했을 때)

사용자가 `.hwpx`를 첨부한 경우, 이 스킬은 아래 순서를 **기본값**으로 따른다.
이 모드에서는 원본 구조를 최대한 보존하고 텍스트/데이터만 교체한다.

1. **레퍼런스 확보**: 첨부된 HWPX를 기준 문서로 사용
2. **심층 분석/추출**: `analyze_template.py`로 `header.xml`, `section0.xml` 추출
3. **구조 복원**: header 스타일 ID/표 구조/셀 병합/여백/문단 흐름을 최대한 동일하게 유지
4. **요청 반영 재작성**: 사용자가 요구한 텍스트/데이터만 교체하고 구조는 보존
5. **빌드/검증**: `build_hwpx.py` + `validate.py`로 결과 산출 및 무결성 확인
6. **쪽수 가드(필수)**: `page_guard.py`로 레퍼런스 대비 페이지 드리프트 위험 검사

### 99% 근접 복원 기준 (실무 체크리스트)

- `charPrIDRef`, `paraPrIDRef`, `borderFillIDRef` 참조 체계 동일
- 표의 `rowCnt`, `colCnt`, `colSpan`, `rowSpan`, `cellSz`, `cellMargin` 동일
- 문단 순서, 문단 수, 주요 빈 줄/구획 위치 동일
- 페이지/여백/섹션(secPr) 동일
- 변경은 사용자 요청 범위(본문 텍스트, 값, 항목명 등)로 제한

### 쪽수 동일(100%) 필수 기준

- 사용자가 레퍼런스를 제공한 경우 **결과 문서의 최종 쪽수는 레퍼런스와 동일해야 한다**
- 쪽수가 늘어날 가능성이 보이면 먼저 텍스트를 압축/요약해서 기존 레이아웃에 맞춘다
- 사용자 명시 요청 없이 `hp:p`, `hp:tbl`, `rowCnt`, `colCnt`, `pageBreak`, `secPr`를 변경하지 않는다
- `validate.py` 통과만으로 완료 처리하지 않는다. 반드시 `page_guard.py`도 통과해야 한다
- `page_guard.py` 실패 시 결과를 완료로 제출하지 않고, 원인(길이 과다/구조 변경)을 수정 후 재빌드한다
- 가능하면 한글(또는 사용자의 확인) 기준 최종 쪽수 값을 확인하고 레퍼런스와 일치 여부를 재확인한다

### 기본 실행 명령 (첨부 레퍼런스가 있을 때)

```bash
# 1) 레퍼런스 분석 + XML 추출
python3 "$SKILL_DIR/scripts/analyze_template.py" reference.hwpx \
  --extract-header /tmp/ref_header.xml \
  --extract-section /tmp/ref_section.xml

# 2) /tmp/ref_section.xml을 복제해 /tmp/new_section0.xml 작성
#    (구조 유지, 텍스트/데이터만 요청에 맞게 수정)

# 3) 복원 빌드
python3 "$SKILL_DIR/scripts/build_hwpx.py" \
  --header /tmp/ref_header.xml \
  --section /tmp/new_section0.xml \
  --output output/result.hwpx

# 4) 검증
python3 "$SKILL_DIR/scripts/validate.py" output/result.hwpx

# 5) 쪽수 드리프트 가드 (필수)
python3 "$SKILL_DIR/scripts/page_guard.py" \
  --reference reference.hwpx \
  --output output/result.hwpx
```

## 0. 환경 점검 (스킬 시작 시 1회, 필수)

> 이 스킬은 비개발자 사용자를 위해 배포된다. 따라서 의존성 누락 시 사용자에게 `pip install`을 요구하지 말고, 아래 절차로 **스킬이 직접 설치**한다.

본격적인 빌드 작업 전에 다음을 순서대로 수행한다:

1. **필수 의존성 확인 (`lxml`)**
   ```bash
   python3 -c "import lxml" 2>/dev/null
   ```
   성공하면 다음 단계로. 실패하면 2번으로 이동.

2. **자동 설치 (실패 시 1회 안내 후 진행)**
   사용자에게 한 줄로 안내한다: "한글파일 스킬 첫 사용 — 의존성 자동 설치 중입니다(약 10~30초)…"
   그 다음 다음 명령을 실행한다:
   ```bash
   python3 -m pip install --user --quiet -r "$SKILL_DIR/requirements.txt"
   ```
   설치 후 다시 `python3 -c "import lxml"`로 확인. 여전히 실패하면 사용자에게 환경 문제(파이썬/네트워크/권한)를 보고하고 작업을 중단한다.

3. **텍스트 추출(`text_extract.py`)이 필요한 요청에 한해서만 `python-hwpx` 확인**
   `requirements.txt`에 이미 포함되어 있어 2번에서 함께 설치된다. 일반적인 보고서 생성 흐름에서는 별도 확인이 불필요하다.

4. **재확인 후 본 작업 계속**
   환경 점검이 한 번 통과하면 같은 세션에서는 다시 점검하지 않는다(이미 import 가능 상태).

> 참고: 설치 명령은 `--user` 플래그로 사용자 홈에 설치하므로 시스템 권한이 필요 없고, `--quiet`로 출력을 최소화한다. `requirements.txt`가 패키지 목록의 단일 출처(single source of truth)다.

## 환경

`SKILL_DIR`는 이 SKILL.md가 위치한 디렉터리의 절대 경로다. 설치 방식에 따라 다음과 같이 결정한다:

| 설치 방식 | `SKILL_DIR` 값 |
|---|---|
| 사용자 전역 설치 | `$HOME/.claude/skills/hwpx` |
| 프로젝트 전용 설치 | `$(pwd)/.claude/skills/hwpx` (프로젝트 루트 기준) |
| 리포 클론 직접 사용 | `$(pwd)` (리포 루트 자체가 곧 스킬) |

Claude agent는 이 SKILL.md를 읽어 들인 실제 위치를 기준으로 `SKILL_DIR`를 자연스럽게 결정한다. 아래 모든 예시의 `$SKILL_DIR`는 이 값으로 치환해 사용한다.

Python 명령은 현재 agent 환경에서 사용 가능한 `python3`로 실행한다. 필수 의존성은 위 "0. 환경 점검" 절차로 자동 처리되므로, 사용자가 별도로 `pip install`할 필요는 없다. 의존성 목록의 정의는 `requirements.txt`에 있다.

## 결과 저장 위치 (모든 출력에 적용 — 필수)

이 스킬이 생성하는 **모든 `.hwpx` 결과물**은 사용자의 **현재 작업 디렉터리(`$(pwd)`) 아래 `output/` 폴더**에 저장한다.

- 예: `--output output/result.hwpx`, `--output output/2026_보고서.hwpx`
- `output/` 폴더가 없으면 `build_hwpx.py`(및 `office/pack.py`)가 **자동 생성**한다 — 사용자에게 미리 만들도록 요구하지 않는다.
- 파일명은 보고서 내용을 식별할 수 있는 한국어/영문 짧은 이름으로 명명한다(예: `output/채용공고계획.hwpx`).
- 사용자가 명시적으로 다른 경로(예: 데스크탑, 특정 폴더)를 지정한 경우에만 그 경로를 우선한다.
- 중간 산출물(임시 section XML 등)은 `output/` 대신 `tempfile`/`/tmp`에 두고 사용 후 삭제한다.

## 디렉토리 구조

```
hwpx/
├── SKILL.md                              # 이 파일
├── scripts/
│   ├── office/
│   │   ├── unpack.py                     # HWPX → 디렉토리 (XML pretty-print)
│   │   └── pack.py                       # 디렉토리 → HWPX
│   ├── build_hwpx.py                     # 템플릿 + XML → .hwpx 조립 (핵심)
│   ├── analyze_template.py               # HWPX 심층 분석 (레퍼런스 기반 생성용)
│   ├── table_builder.py                  # 표 템플릿 → XML 생성
│   ├── preview_table.py                  # 표 템플릿 + 샘플 데이터 → 미리보기 .hwpx
│   ├── validate.py                       # HWPX 구조 검증
│   ├── page_guard.py                     # 레퍼런스 대비 페이지 드리프트 위험 검사
│   └── text_extract.py                   # 텍스트 추출 (python-hwpx 필요)
├── templates/
│   ├── base/                             # 베이스 템플릿 (Skeleton 기반)
│   │   ├── mimetype, META-INF/*, version.xml, settings.xml, Preview/*
│   │   └── Contents/ (header.xml, section0.xml, content.hpf)
│   ├── report/                           # 보고서 오버레이 (header.xml, section0.xml)
│   └── tables/                           # 표 템플릿 (table_builder.py 사용)
│       ├── basic.xml                     # 기본 표 (4열: 연번/구분/내용/비고)
│       ├── status.xml                    # 현황표 (4열: 번호/항목/추진현황/진행률)
│       ├── budget.xml                    # 예산표 (5열, 합계행 포함)
│       ├── schedule.xml                  # 일정표 (4열: 기간/추진내용/세부사항/담당)
│       └── checklist.xml                 # 점검표 (5열: 번호/점검항목/담당/확인/비고)
├── assets/
│   ├── report-template.hwpx              # report 템플릿 시각 기준 샘플 (런타임 미사용)
│   └── all_tables_preview.hwpx           # 표 템플릿(tables/) 5종 종합 미리보기 (런타임 미사용)
└── references/
    └── hwpx-format.md                    # OWPML XML 요소 레퍼런스
```

`assets/`는 최종 문서 생성에 직접 투입되는 런타임 템플릿이 아니라, 필요할 때 한글에서 열어 레이아웃을 확인하는 기준 샘플을 보관하는 위치다.

---

## 모드 2: 템플릿 기반 새 문서 생성 (레퍼런스 파일이 없을 때)

### 템플릿 자동 라우팅 (필수 절차)

이 모드에서는 사용자 요청의 성격을 보고 `--template` 값을 **스킬이 자동 선택**한다.
명확한 단서가 없거나 보고서/공문/추진계획 등 일반적인 문서 작성 요청이면 `report`를 사용한다.

| 템플릿 | 용도 | 트리거 키워드 |
|--------|------|---------------|
| `report` | 보고서·공문·추진계획 등 모든 정형 문서 (기본값) | "보고서", "추진계획", "추진현황", "업무보고", "결과보고", "공문", "결재", "장관/시장/원장 보고", 별도 단서 없음 |
| `base` | 보고서 양식이 불필요한 문서(메모, 표만 있는 문서 등) | "메모", "단순 문서", "표만", "스켈레톤" |

라우팅 규칙:

- 사용자가 명시적으로 템플릿 이름을 적은 경우 그 값을 따른다(예: "base로 만들어줘").
- 모호한 경우 한 차례에 한해 사용자에게 확인한 뒤 진행하고, 결정값을 그대로 사용한다.
- 결정한 템플릿을 작업 시작 시점에 사용자에게 한 줄로 알린다(예: "→ template=report 사용").

### 핵심 원칙: 템플릿은 양식이지 콘텐츠 틀이 아니다

- **header.xml** → 그대로 사용한다 (스타일 정의: 폰트, 크기, 색상, 문단 간격 등)
- **section0.xml** → secPr(페이지 설정) 첫 문단만 가져오고, **본문은 내용에 맞게 새로 작성**한다
- 템플릿의 section0.xml은 "어떤 스타일 ID를 어떤 용도로 쓰는지" 보여주는 **서식 참고용**이다
- 템플릿의 문단 수, 표 행 수, 섹션 수를 그대로 따르지 않는다
- 내용이 5개 섹션이면 5개를 만들고, 표가 10행이면 10행으로 만든다
- **본문 최소 글자 수**: □/❍ 항목 등 본문 텍스트는 한 줄에 20자 이상 작성한다. 너무 짧은 문구는 보고서 품질을 떨어뜨린다
- **표 빈 셀 금지**: 표의 모든 셀에 반드시 내용을 채운다. 빈 문자열("") 대신 "-" 또는 적절한 값 사용
- **표 열 폭 조정**: 각 셀에 들어가는 글자 수를 고려하여 열 너비를 조정한다. 내용이 긴 열은 넓게, 짧은 열(번호, 확인 등)은 좁게 설정
- **표 밀도 제한**: 한 페이지에 표는 최대 1개만 배치한다. 표가 여러 개 필요하면 본문 텍스트로 충분히 간격을 두거나 페이지를 분리한다

### 흐름

1. **템플릿 자동 선택** — 위 "템플릿 자동 라우팅" 표에 따라 `report` / `base` 중 결정
2. **header.xml 확인** — 사용 가능한 스타일 ID(charPr, paraPr, borderFill) 파악
3. **section0.xml 새로 작성** — secPr은 템플릿에서 복사, 본문은 내용 분량에 맞게 자유롭게 구성
4. **build_hwpx.py로 빌드**
5. **validate.py로 검증**

> 원칙: 사용자가 레퍼런스 HWPX를 제공한 경우에는 이 워크플로우 대신 상단의 "기본 동작 모드(레퍼런스 복원 우선)"를 사용한다.

### 기본 사용법

```bash
# 빈 문서 (base 템플릿)
python3 "$SKILL_DIR/scripts/build_hwpx.py" --output output/result.hwpx

# 보고서 템플릿 사용
python3 "$SKILL_DIR/scripts/build_hwpx.py" --template report --output output/result.hwpx

# 커스텀 section0.xml 오버라이드
python3 "$SKILL_DIR/scripts/build_hwpx.py" --template report --section my_section0.xml --output output/result.hwpx

# header도 오버라이드
python3 "$SKILL_DIR/scripts/build_hwpx.py" --header my_header.xml --section my_section0.xml --output output/result.hwpx

# 메타데이터 설정
python3 "$SKILL_DIR/scripts/build_hwpx.py" --template report --section my.xml \
  --title "제목" --creator "작성자" --output output/result.hwpx
```

### 실전 패턴: 템플릿에서 secPr만 복사 → 본문 자유 작성 → 빌드

**Step 1**: 템플릿의 section0.xml에서 secPr 포함 첫 문단을 복사 (페이지 설정)
**Step 2**: 나머지 본문은 내용 분량에 맞게 자유롭게 작성 (charPrIDRef/paraPrIDRef만 header.xml 참조)
**Step 3**: 빌드

```bash
# 1. section0.xml을 임시파일로 작성
SECTION=$(mktemp /tmp/section0_XXXX.xml)
cat > "$SECTION" << 'XMLEOF'
<?xml version='1.0' encoding='UTF-8'?>
<hs:sec xmlns:hp="http://www.hancom.co.kr/hwpml/2011/paragraph"
        xmlns:hs="http://www.hancom.co.kr/hwpml/2011/section">
  <!-- secPr 포함 첫 문단 (템플릿 section0.xml에서 복사) -->
  <!-- ... -->

  <!-- ▼ 여기서부터 내용에 맞게 자유 작성 — 문단/표/섹션 수에 제한 없음 -->
  <hp:p id="1000000002" paraPrIDRef="0" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">
    <hp:run charPrIDRef="0">
      <hp:t>본문 내용을 자유롭게 작성</hp:t>
    </hp:run>
  </hp:p>
  <!-- 필요한 만큼 문단, 표, 빈 줄 추가 -->
</hs:sec>
XMLEOF

# 2. 빌드
python3 "$SKILL_DIR/scripts/build_hwpx.py" --section "$SECTION" --output output/result.hwpx

# 3. 정리
rm -f "$SECTION"
```

---

## section0.xml 작성 가이드

### 필수 구조

section0.xml의 첫 문단(`<hp:p>`)의 첫 런(`<hp:run>`)에 반드시 `<hp:secPr>`과 `<hp:colPr>` 포함:

```xml
<hp:p id="1000000001" paraPrIDRef="0" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">
  <hp:run charPrIDRef="0">
    <hp:secPr ...>
      <!-- 페이지 크기, 여백, 각주/미주 설정 등 -->
    </hp:secPr>
    <hp:ctrl>
      <hp:colPr id="" type="NEWSPAPER" layout="LEFT" colCount="1" sameSz="1" sameGap="0"/>
    </hp:ctrl>
  </hp:run>
  <hp:run charPrIDRef="0"><hp:t/></hp:run>
</hp:p>
```

**Tip**: `templates/base/Contents/section0.xml` 의 첫 문단을 그대로 복사하면 된다.

### 문단

```xml
<hp:p id="고유ID" paraPrIDRef="문단스타일ID" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">
  <hp:run charPrIDRef="글자스타일ID">
    <hp:t>텍스트 내용</hp:t>
  </hp:run>
</hp:p>
```

### 빈 줄

```xml
<hp:p id="고유ID" paraPrIDRef="0" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">
  <hp:run charPrIDRef="0"><hp:t/></hp:run>
</hp:p>
```

### 서식 혼합 런 (한 문단에 여러 스타일)

```xml
<hp:p id="고유ID" paraPrIDRef="0" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">
  <hp:run charPrIDRef="0"><hp:t>일반 텍스트 </hp:t></hp:run>
  <hp:run charPrIDRef="7"><hp:t>볼드 텍스트</hp:t></hp:run>
  <hp:run charPrIDRef="0"><hp:t> 다시 일반</hp:t></hp:run>
</hp:p>
```

### 표 작성법

```xml
<hp:p id="고유ID" paraPrIDRef="0" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0">
  <hp:run charPrIDRef="0">
    <hp:tbl id="고유ID" zOrder="0" numberingType="TABLE" textWrap="TOP_AND_BOTTOM"
            textFlow="BOTH_SIDES" lock="0" dropcapstyle="None" pageBreak="CELL"
            repeatHeader="0" rowCnt="행수" colCnt="열수" cellSpacing="0"
            borderFillIDRef="3" noAdjust="0">
      <hp:sz width="42520" widthRelTo="ABSOLUTE" height="전체높이" heightRelTo="ABSOLUTE" protect="0"/>
      <hp:pos treatAsChar="1" affectLSpacing="0" flowWithText="1" allowOverlap="0"
              holdAnchorAndSO="0" vertRelTo="PARA" horzRelTo="COLUMN" vertAlign="TOP"
              horzAlign="LEFT" vertOffset="0" horzOffset="0"/>
      <hp:outMargin left="0" right="0" top="0" bottom="0"/>
      <hp:inMargin left="0" right="0" top="0" bottom="0"/>
      <hp:tr>
        <hp:tc name="" header="0" hasMargin="0" protect="0" editable="0" dirty="1" borderFillIDRef="4">
          <hp:subList id="" textDirection="HORIZONTAL" lineWrap="BREAK" vertAlign="CENTER"
                     linkListIDRef="0" linkListNextIDRef="0" textWidth="0" textHeight="0"
                     hasTextRef="0" hasNumRef="0">
            <hp:p paraPrIDRef="21" styleIDRef="0" pageBreak="0" columnBreak="0" merged="0" id="고유ID">
              <hp:run charPrIDRef="9"><hp:t>헤더 셀</hp:t></hp:run>
            </hp:p>
          </hp:subList>
          <hp:cellAddr colAddr="0" rowAddr="0"/>
          <hp:cellSpan colSpan="1" rowSpan="1"/>
          <hp:cellSz width="열너비" height="행높이"/>
          <hp:cellMargin left="0" right="0" top="0" bottom="0"/>
        </hp:tc>
        <!-- 나머지 셀... -->
      </hp:tr>
    </hp:tbl>
  </hp:run>
</hp:p>
```

### 표 크기 계산

- **A4 본문폭**: 42520 HWPUNIT = 59528(용지) - 8504×2(좌우여백)
- **열 너비 합 = 본문폭** (42520)
- 예: 3열 균등 → 14173 + 14173 + 14174 = 42520
- 예: 2열 (라벨:내용 = 1:4) → 8504 + 34016 = 42520
- **행 높이**: 셀당 보통 2400~3600 HWPUNIT

### ID 규칙

- 문단 id: `1000000001`부터 순차 증가
- 표 id: `1000000099` 등 별도 범위 사용 권장
- 모든 id는 문서 내 고유해야 함

---

## 표 템플릿 시스템 (table_builder.py)

`templates/tables/` 에 저장된 표 양식을 기반으로, 데이터만 넣으면 완성된 표 XML을 생성한다.

### 사용 가능한 템플릿

| 템플릿 | 열 구성 | 합계행 | 용도 |
|--------|---------|--------|------|
| `basic` | 연번/구분/내용/비고 (4열) | - | 범용 표 |
| `status` | 번호/항목/추진현황/진행률 (4열) | - | 현황 보고 |
| `budget` | 연번/사업명/내용/예산/비고 (5열) | ✓ | 예산 내역 |
| `schedule` | 기간/추진내용/세부사항/담당 (4열) | - | 일정 계획 |
| `checklist` | 번호/점검항목/담당/확인/비고 (5열) | - | 점검·체크리스트 |

### Python API 사용법

```python
import sys; sys.path.insert(0, f"{SKILL_DIR}/scripts")
from table_builder import build_table, build_table_paragraph

# 표 XML만 생성 (hp:tbl)
table_xml = build_table(
    template="basic",
    data=[["1", "교원연수", "AI 직무연수 운영", "상반기"],
          ["2", "수업모델", "수업안 개발", "5개 교과"]],
    headers=["연번", "구분", "내용", "비고"],  # 선택: 헤더 텍스트 오버라이드
    start_id=1000000050,
)

# hp:p로 감싼 버전 (section0.xml에 바로 삽입 가능)
para_xml = build_table_paragraph(
    template="budget",
    data=[["1", "인건비", "개발자 3명", "500", ""],
          ["2", "장비", "서버 구매", "200", ""]],
    summary=["", "", "합  계", "700", ""],  # 합계행 (budget 전용)
    start_id=1000000050,
)
```

### CLI 사용법

```bash
# 템플릿 목록 확인
python3 "$SKILL_DIR/scripts/table_builder.py" --list

# 표 XML 생성
python3 "$SKILL_DIR/scripts/table_builder.py" --template basic \
  --data '[["1","교원연수","AI 직무연수","상반기"]]' \
  --start-id 1000000050

# hp:p 포함 출력 (--paragraph)
python3 "$SKILL_DIR/scripts/table_builder.py" --template basic \
  --data '[["1","A","B","C"]]' --paragraph --start-id 1000000050
```

### 핵심 규칙

- **start_id**: 문서 내 다른 요소와 겹치지 않아야 한다. 표마다 다른 범위 사용
- **열 수 일치**: data 각 행의 원소 수는 템플릿 열 수와 일치해야 한다
- **합계행**: `summary` 파라미터는 budget처럼 summary row가 정의된 템플릿에서만 동작
- **헤더 오버라이드**: `headers` 미지정 시 템플릿의 기본 헤더 텍스트 사용
- **빈 셀 금지**: 모든 셀에 반드시 내용을 채운다. 빈 문자열("") 대신 "-" 또는 적절한 값 사용

### 표 미리보기 (preview_table.py)

표 템플릿에 샘플 데이터를 채워 한글에서 바로 열어볼 수 있는 `.hwpx`를 생성한다 (디자인 확인용).

```bash
# 사용 가능한 표 템플릿 목록
python3 "$SKILL_DIR/scripts/preview_table.py" --list

# 단일 템플릿 미리보기
python3 "$SKILL_DIR/scripts/preview_table.py" budget --output output/budget_preview.hwpx

# 전체 템플릿을 한 번에 미리보기 생성
python3 "$SKILL_DIR/scripts/preview_table.py" --all
```

---

## header.xml 수정 가이드

### 커스텀 스타일 추가 방법

1. `templates/base/Contents/header.xml` 복사
2. 필요한 charPr/paraPr/borderFill 추가
3. 각 그룹의 `itemCnt` 속성 업데이트

### charPr 추가 예시 (볼드 14pt)

```xml
<hh:charPr id="8" height="1400" textColor="#000000" shadeColor="none"
           useFontSpace="0" useKerning="0" symMark="NONE" borderFillIDRef="2">
  <hh:fontRef hangul="1" latin="1" hanja="1" japanese="1" other="1" symbol="1" user="1"/>
  <hh:ratio hangul="100" latin="100" hanja="100" japanese="100" other="100" symbol="100" user="100"/>
  <hh:spacing hangul="0" latin="0" hanja="0" japanese="0" other="0" symbol="0" user="0"/>
  <hh:relSz hangul="100" latin="100" hanja="100" japanese="100" other="100" symbol="100" user="100"/>
  <hh:offset hangul="0" latin="0" hanja="0" japanese="0" other="0" symbol="0" user="0"/>
  <hh:bold/>
  <hh:underline type="NONE" shape="SOLID" color="#000000"/>
  <hh:strikeout shape="NONE" color="#000000"/>
  <hh:outline type="NONE"/>
  <hh:shadow type="NONE" color="#C0C0C0" offsetX="10" offsetY="10"/>
</hh:charPr>
```

### 폰트 참조 체계

- `fontRef` 값은 `fontfaces`에 정의된 font id
- `hangul="0"` → 굴림, `"1"` → 궁서, `"2"` → 맑은고딕, `"3"` → 함초롬돋움
- `hangul="4"` → 휴먼명조, `"5"` → HY헤드라인M, `"7"` → 한양중고딕, `"8"` → 바탕
- 7개 언어(HANGUL/LATIN/HANJA/JAPANESE/OTHER/SYMBOL/USER) 모두 동일하게 설정

---

## report 템플릿 스타일 ID 맵

> `templates/report/header.xml` 기준. `analyze_template.py`로 확인 가능.

### 본문 스타일 (section0.xml 작성 시 사용)

| 용도 | charPrIDRef | paraPrIDRef | 설명 |
|------|-------------|-------------|------|
| 소제목 (1. 2. 3.) | 10 | 2 | 16pt HY헤드라인M |
| 소제목 후 빈줄 | 12 | 2 | 4pt 스페이서 |
| □/❍ 본문 항목 | 25 | 22 | 13pt 휴먼명조 |
| ※ 참고 사항 | 20 | 21 | 13pt 한양중고딕 |
| 섹션 구분 빈줄 | 13 | 2 | 11pt 한양중고딕 |
| 제목 아래 빈줄 | 9 | 14 | (양식 고정) |

### 표 스타일 (table_builder 템플릿에서 사용)

| 용도 | charPrIDRef | paraPrIDRef | borderFillIDRef |
|------|-------------|-------------|-----------------|
| 표 헤더 셀 | 26 (12pt 휴먼명조 bold) | 18 (CENTER) | 9 (회색 #D9D9D9) |
| 표 본문 셀 | 27 (12pt 휴먼명조) | 18 (CENTER) 또는 22 (JUSTIFY) | 10 (테두리만) |
| 표 합계행 | 26 | 18 | 9 |
| 표 컨테이너 | - | - | 4 (SOLID 테두리) |

### 헤더 영역 (secPr + 제목, 양식에서 그대로 복사)

| 용도 | charPrIDRef | paraPrIDRef | 비고 |
|------|-------------|-------------|------|
| 날짜·부서 문단 | 21, 9, 22, 18 | 19 | secPr 포함, 다중 run |
| 제목 drawText | 24 (내부: 25) | 2 | rect 도형 안 텍스트 |

---

## 보조 워크플로우: 기존 문서 편집 (unpack → Edit → pack)

```bash
# 1. HWPX → 디렉토리 (XML pretty-print)
python3 "$SKILL_DIR/scripts/office/unpack.py" document.hwpx ./unpacked/

# 2. XML 직접 편집 (agent가 파일 편집 도구로)
#    본문: ./unpacked/Contents/section0.xml
#    스타일: ./unpacked/Contents/header.xml

# 3. 다시 HWPX로 패키징
python3 "$SKILL_DIR/scripts/office/pack.py" ./unpacked/ output/edited.hwpx

# 4. 검증
python3 "$SKILL_DIR/scripts/validate.py" output/edited.hwpx
```

---

## 보조 워크플로우: 읽기/텍스트 추출

```bash
# 순수 텍스트
python3 "$SKILL_DIR/scripts/text_extract.py" document.hwpx

# 테이블 포함
python3 "$SKILL_DIR/scripts/text_extract.py" document.hwpx --include-tables

# 마크다운 형식
python3 "$SKILL_DIR/scripts/text_extract.py" document.hwpx --format markdown
```

### Python API

```python
from hwpx import TextExtractor
with TextExtractor("document.hwpx") as ext:
    text = ext.extract_text(include_nested=True, object_behavior="nested")
    print(text)
```

---

## 보조 워크플로우: 검증

```bash
python3 "$SKILL_DIR/scripts/validate.py" document.hwpx
```

검증 항목: ZIP 유효성, 필수 파일 존재, mimetype 내용/위치/압축방식, XML well-formedness

---

## 레퍼런스 기반 문서 생성 상세 (모드 1 상세 가이드)

> 상단 "모드 1: 레퍼런스 복원"의 상세 실행 가이드이다.

### 사용법

```bash
# 1. 심층 분석 (구조 청사진 출력)
python3 "$SKILL_DIR/scripts/analyze_template.py" reference.hwpx

# 2. header.xml과 section0.xml을 추출하여 참고용으로 보관
python3 "$SKILL_DIR/scripts/analyze_template.py" reference.hwpx \
  --extract-header /tmp/ref_header.xml \
  --extract-section /tmp/ref_section.xml

# 3. 분석 결과를 보고 새 section0.xml 작성
#    - 동일한 charPrIDRef, paraPrIDRef 사용
#    - 동일한 테이블 구조 (열 수, 열 너비, 행 수, rowSpan/colSpan)
#    - 동일한 borderFillIDRef, cellMargin

# 4. 추출한 header.xml + 새 section0.xml로 빌드
python3 "$SKILL_DIR/scripts/build_hwpx.py" \
  --header /tmp/ref_header.xml \
  --section /tmp/new_section0.xml \
  --output output/result.hwpx

# 5. 검증
python3 "$SKILL_DIR/scripts/validate.py" output/result.hwpx

# 6. 쪽수 드리프트 가드 (필수)
python3 "$SKILL_DIR/scripts/page_guard.py" \
  --reference reference.hwpx \
  --output output/result.hwpx
```

### 분석 출력 항목

| 항목 | 설명 |
|------|------|
| 폰트 정의 | hangul/latin 폰트 매핑 |
| borderFill | 테두리 타입/두께 + 배경색 (각 면별 상세) |
| charPr | 글꼴 크기(pt), 폰트명, 색상, 볼드/이탤릭/밑줄/취소선, fontRef |
| paraPr | 정렬, 줄간격, 여백(left/right/prev/next/intent), heading, borderFillIDRef |
| 문서 구조 | 페이지 크기, 여백, 페이지 테두리, 본문폭 |
| 본문 상세 | 모든 문단의 id/paraPr/charPr + 텍스트 내용 |
| 표 상세 | 행×열, 열너비 배열, 셀별 span/margin/borderFill/vertAlign + 내용 |

### 핵심 원칙

- **charPrIDRef/paraPrIDRef를 그대로 사용**: 추출한 header.xml의 스타일 ID를 변경하지 말 것
- **열 너비 합계 = 본문폭**: 분석 결과의 열너비 배열을 그대로 복제
- **rowSpan/colSpan 패턴 유지**: 분석된 셀 병합 구조를 정확히 재현
- **cellMargin 보존**: 분석된 셀 여백 값을 동일하게 적용
- **페이지 증가 금지**: 사용자 명시 승인 없이 결과 쪽수를 늘리지 말 것
- **치환 우선 편집**: 새 문단/표 추가보다 기존 텍스트 노드 치환을 우선할 것

---

## 스크립트 요약

| 스크립트 | 용도 |
|----------|------|
| `scripts/build_hwpx.py` | **핵심** — 템플릿 + XML → HWPX 조립 |
| `scripts/table_builder.py` | 표 템플릿 → 데이터 주입 → 표 XML 생성 |
| `scripts/preview_table.py` | 표 템플릿 + 샘플 데이터 → 미리보기 .hwpx 생성 |
| `scripts/analyze_template.py` | HWPX 심층 분석 (레퍼런스 기반 생성의 청사진) |
| `scripts/office/unpack.py` | HWPX → 디렉토리 (XML pretty-print) |
| `scripts/office/pack.py` | 디렉토리 → HWPX (mimetype first) |
| `scripts/validate.py` | HWPX 파일 구조 검증 |
| `scripts/page_guard.py` | 레퍼런스 대비 페이지 드리프트 위험 검사 (필수 게이트) |
| `scripts/text_extract.py` | HWPX 텍스트 추출 (python-hwpx 필요) |

## 단위 변환

| 값 | HWPUNIT | 의미 |
|----|---------|------|
| 1pt | 100 | 기본 단위 |
| 10pt | 1000 | 기본 글자크기 |
| 1mm | 283.5 | 밀리미터 |
| 1cm | 2835 | 센티미터 |
| A4 폭 | 59528 | 210mm |
| A4 높이 | 84186 | 297mm |
| 좌우여백 | 8504 | 30mm |
| 본문폭 | 42520 | 150mm (A4-좌우여백) |

## Critical Rules

1. **HWPX만 지원**: `.hwp`(바이너리) 파일은 지원하지 않는다. 사용자가 `.hwp` 파일을 제공하면 **한글 오피스에서 `.hwpx`로 다시 저장**하도록 안내할 것. (파일 → 다른 이름으로 저장 → 파일 형식: HWPX)
2. **secPr 필수**: section0.xml 첫 문단의 첫 run에 반드시 secPr + colPr 포함
3. **mimetype 순서**: HWPX 패키징 시 mimetype은 첫 번째 ZIP 엔트리, ZIP_STORED
4. **네임스페이스 보존**: XML 편집 시 `hp:`, `hs:`, `hh:`, `hc:` 접두사 유지
5. **itemCnt 정합성**: header.xml의 charProperties/paraProperties/borderFills itemCnt가 실제 자식 수와 일치
6. **ID 참조 정합성**: section0.xml의 charPrIDRef/paraPrIDRef가 header.xml 정의와 일치
7. **Python 환경**: 현재 환경의 `python3` 사용 (`lxml` 필요, 일부 보조 스크립트는 `hwpx` 패키지 필요)
8. **검증**: 생성 후 반드시 `validate.py`로 무결성 확인
9. **레퍼런스**: 상세 XML 구조는 `$SKILL_DIR/references/hwpx-format.md` 참조
10. **build_hwpx.py 우선**: 새 문서 생성은 build_hwpx.py 사용 (python-hwpx API 직접 호출 지양)
11. **빈 줄**: `<hp:t/>` 사용 (self-closing tag)
12. **레퍼런스 우선 강제**: 사용자가 HWPX를 첨부하면 반드시 `analyze_template.py` + 추출 XML 기반으로 복원/재작성할 것
13. **쪽수 동일 필수**: 레퍼런스 기반 작업에서는 최종 결과의 쪽수를 레퍼런스와 동일하게 유지할 것
14. **무단 페이지 증가 금지**: 사용자 명시 요청/승인 없이 쪽수 증가를 유발하는 구조 변경 금지
15. **구조 변경 제한**: 사용자 요청이 없는 한 문단/표의 추가·삭제·분할·병합 금지 (치환 중심 편집)
16. **page_guard 필수 통과**: `validate.py`와 별개로 `page_guard.py`를 반드시 통과해야 완료 처리
17. **결과 저장 위치**: 모든 `.hwpx` 결과는 `$(pwd)/output/` 폴더에 저장한다. 폴더 미존재 시 자동 생성. 사용자가 다른 경로를 명시한 경우만 예외. ([결과 저장 위치] 섹션 참조)
