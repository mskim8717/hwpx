# hwpx — 한글(HWPX) 보고서 자동화 스킬 for Claude Code

OWPML 표준의 .hwpx 보고서를 **XML 직접 작성** 방식으로 생성·읽기·편집하는 Claude Code 스킬입니다. python-hwpx API의 서식 버그를 우회하며, 보고서·표·페이지 레이아웃을 세밀하게 제어할 수 있습니다.

## 주요 기능

- **report 양식 기반 보고서 생성**: `templates/report/`에 정의된 단일 보고서 양식으로 출력. 본문에 맞춰 section XML만 새로 작성하면 됩니다.
- **표 템플릿 시스템**: `basic`, `status`, `budget`, `schedule`, `checklist` 5종 + 데이터 주입식 생성
- **읽기/편집 도구**: 텍스트 추출, HWPX ↔ 디렉토리 unpack/pack, ZIP/XML 구조 검증

> 사용자가 .hwpx 파일을 첨부해도 그 구조를 그대로 복원하지는 않습니다. 첨부 파일은 본문 내용 참고용으로만 사용되고, 출력은 항상 report 양식으로 생성됩니다. 원본 양식 그대로의 복원이 필요하면 한글 오피스에서 직접 편집해주세요.

## 설치

> Claude Code 등 agent에게 GitHub URL(`https://github.com/mskim8717/hwpx`)만 주고 "이거 설치해줘"라고 부탁하면, agent가 아래 절차를 따라 설치합니다. 사용자가 직접 실행해도 동일합니다.

### 사용자 전역 스킬로 설치 (모든 프로젝트에서 사용 — 권장)

```bash
mkdir -p ~/.claude/skills
git clone --depth 1 https://github.com/mskim8717/hwpx.git ~/.claude/skills/hwpx
python3 -m pip install --user -r ~/.claude/skills/hwpx/requirements.txt
```

### 현재 프로젝트 전용 스킬로 설치

```bash
mkdir -p .claude/skills
git clone --depth 1 https://github.com/mskim8717/hwpx.git .claude/skills/hwpx
python3 -m pip install --user -r .claude/skills/hwpx/requirements.txt
```

설치 후 Claude Code를 재시작하면 "한글파일 생성", "보고서 만들어줘", ".hwpx 작성" 같은 요청 시 자동으로 스킬이 호출됩니다.

## 프롬프트 작성 Best Practice

좋은 HWPX 결과물을 얻으려면 "무엇을 써줘"보다 **문서 목적, 보고 대상, 포함할 항목, 표 구성, 분량, 파일명**을 함께 알려주는 것이 좋습니다.

### 기본 템플릿

```text
다음 조건으로 한글파일(.hwpx) 보고서를 작성해줘.

1. 문서 종류:
2. 보고 대상:
3. 작성 목적:
4. 핵심 내용:
5. 포함할 표:
6. 분량과 톤:
7. 파일명:

생성 후 HWPX 구조 검증까지 해줘.
```

### 예시 1: 새 보고서 생성

```text
2026년 1학기 교원 AI 활용 역량강화 추진계획 보고서를 한글파일(.hwpx)로 작성해줘.

보고 대상은 교육지원청 과장이고, 목적은 관내 초중고 교원의 AI 수업 활용 역량을 높이기 위한 연수 계획 보고야.
문서는 보고자료 형식으로 작성하고, 다음 항목을 포함해줘.

1. 추진 배경: 디지털 기반 수업 전환, 교원 AI 활용 격차, 학교 현장 지원 필요성
2. 추진 방향: 실습 중심 연수, 교과별 수업 사례 개발, 사후 컨설팅
3. 세부 추진 계획: 기초 연수, 심화 연수, 수업 나눔회, 현장 컨설팅
4. 기대 효과: 수업 설계 역량 향상, 학생 맞춤형 학습 지원, 우수 사례 확산

표는 일정표 1개를 포함하고, 기간/추진내용/세부사항/담당 열로 구성해줘.
전체 분량은 A4 2쪽 이내로 하고, 공공기관 보고서처럼 간결하고 격식 있게 써줘.
파일명은 ai_teacher_training_plan.hwpx로 저장해줘.
생성 후 validate.py로 구조 검증까지 해줘.
```

### 예시 2: 표 중심 현황 보고서

```text
지역 돌봄교실 운영 현황 보고서를 한글파일(.hwpx)로 작성해줘.

보고 대상은 시청 교육협력과 팀장이고, 목적은 2026년 상반기 돌봄교실 운영 현황과 개선 필요사항을 보고하는 거야.
본문은 1. 운영 개요, 2. 주요 현황, 3. 문제점, 4. 개선 계획 순서로 작성해줘.

다음 표 2개를 넣어줘.
- 현황표: 번호/항목/추진현황/진행률
- 예산표: 연번/사업명/내용/예산/비고, 마지막 행에 합계 포함

주요 데이터는 아래를 반영해줘.
- 운영 학교: 18개교
- 참여 학생: 642명
- 전담 인력: 36명
- 만족도 조사: 평균 4.3점
- 예산 집행률: 71%
- 개선 필요사항: 대체 인력 확보, 방학 중 프로그램 다양화, 안전관리 매뉴얼 보완

표의 빈 셀은 만들지 말고, 값이 없으면 "-"로 채워줘.
전체 톤은 행정 보고서 문체로 쓰고, 파일명은 care_class_status_report.hwpx로 저장해줘.
```

### 예시 3: 기존 HWPX 일부 수정

```text
첨부한 기존 한글파일(.hwpx)을 수정해줘.

수정 범위는 아래로 제한해줘.
1. 문서 제목을 "2026년 디지털 교육 기반 구축 결과보고"로 변경
2. 본문 중 "2025년"은 모두 "2026년"으로 변경
3. 예산표의 총액을 185,000천원으로 수정
4. 마지막 문단에 "향후 학교별 운영 실적을 분기별로 점검하고 개선 사항을 보완할 예정임"을 추가

수정 후 validate.py로 구조 검증까지 해주고, 파일명은 digital_education_result_2026.hwpx로 저장해줘.
```

> 이 흐름은 내부적으로 `office/unpack.py` → XML 편집 → `office/pack.py`를 사용합니다. 본문 텍스트 치환 위주의 작은 변경에 적합합니다.

## 업데이트

```bash
# 사용자 전역
git -C ~/.claude/skills/hwpx pull

# 프로젝트 전용
git -C .claude/skills/hwpx pull
```

## 제거

```bash
rm -rf ~/.claude/skills/hwpx        # 사용자 전역
rm -rf .claude/skills/hwpx          # 프로젝트 전용
```

## 의존성

**별도 설치 작업 불필요.** 스킬이 처음 호출될 때 필요한 Python 패키지를 자동으로 확인·설치합니다(약 10~30초). 비개발자 사용자도 추가 명령을 입력할 필요가 없습니다.

내부적으로 사용되는 패키지(참고용):

- `lxml` (필수, 자동 설치)
- `python-hwpx` (텍스트 추출 기능을 쓸 때만 사용, 함께 자동 설치)

오프라인 환경 등 자동 설치가 불가능한 경우에만 미리 수동 설치:

```bash
pip install --user -r requirements.txt
```

## 디렉토리 구조

```
hwpx/                            # 이 리포지토리 = 스킬 본체
├── SKILL.md                     # 스킬 본문 (자세한 사용법)
├── requirements.txt
├── scripts/                     # build_hwpx, table_builder, validate, text_extract 등
├── templates/                   # report/ (보고서 양식), tables/ (표 템플릿), base/ (내부 스켈레톤)
├── assets/                      # 시각 기준 .hwpx 샘플
├── references/                  # OWPML 포맷 참조 문서
├── README.md
└── LICENSE
```

자세한 사용법, XML 작성 가이드, 스타일 ID 맵은 [`SKILL.md`](SKILL.md)를 참조하세요.

## 라이선스

[MIT](LICENSE)
