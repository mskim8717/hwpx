# hwpx — 한글(HWPX) 문서 자동화 스킬 for Claude Code

OWPML 표준의 .hwpx 파일을 **XML 직접 작성** 방식으로 생성·읽기·편집하는 Claude Code 스킬입니다. python-hwpx API의 서식 버그를 우회하며, 보고서·표·페이지 레이아웃을 세밀하게 제어할 수 있습니다.

## 주요 기능

- **레퍼런스 복원 모드**: 사용자가 첨부한 .hwpx의 구조·서식·쪽수를 보존하며 텍스트/데이터만 교체
- **템플릿 기반 생성 모드**: `report`(보고서), `base`(빈 문서) 중 요청 키워드에 따라 자동 라우팅
- **표 템플릿 시스템**: `basic`, `status`, `budget`, `schedule`, `checklist` 5종 + 데이터 주입식 생성
- **검증 도구**: ZIP/XML 구조 검증, 레퍼런스 대비 페이지 드리프트 가드

## 설치

> Claude Code 등 agent에게 GitHub URL(`https://github.com/mskim8717/hwpx`)만 주고 "이거 설치해줘"라고 부탁하면, agent가 아래 절차를 따라 설치합니다. 사용자가 직접 실행해도 동일합니다.

### 사용자 전역 스킬로 설치 (모든 프로젝트에서 사용 — 권장)

```bash
git clone https://github.com/mskim8717/hwpx.git /tmp/hwpx-src
mkdir -p ~/.claude/skills
rm -rf ~/.claude/skills/hwpx
cp -r /tmp/hwpx-src/skills/hwpx ~/.claude/skills/
python3 -m pip install --user -r ~/.claude/skills/hwpx/requirements.txt
rm -rf /tmp/hwpx-src
```

### 현재 프로젝트 전용 스킬로 설치

```bash
git clone https://github.com/mskim8717/hwpx.git /tmp/hwpx-src
mkdir -p .claude/skills
rm -rf .claude/skills/hwpx
cp -r /tmp/hwpx-src/skills/hwpx .claude/skills/
python3 -m pip install --user -r .claude/skills/hwpx/requirements.txt
rm -rf /tmp/hwpx-src
```

설치 후 Claude Code를 재시작하면 "한글파일 생성", "보고서 만들어줘", ".hwpx 작성" 같은 요청 시 자동으로 스킬이 호출됩니다.

## 의존성

**별도 설치 작업 불필요.** 스킬이 처음 호출될 때 필요한 Python 패키지를 자동으로 확인·설치합니다(약 10~30초). 비개발자 사용자도 추가 명령을 입력할 필요가 없습니다.

내부적으로 사용되는 패키지(참고용):

- `lxml` (필수, 자동 설치)
- `python-hwpx` (텍스트 추출 기능을 쓸 때만 사용, 함께 자동 설치)

오프라인 환경 등 자동 설치가 불가능한 경우에만 미리 수동 설치:

```bash
pip install --user -r skills/hwpx/requirements.txt
```

## 디렉토리 구조

```
hwpx/
├── skills/hwpx/
│   ├── SKILL.md                # 스킬 본문 (자세한 사용법)
│   ├── requirements.txt
│   ├── scripts/                # build_hwpx, validate, analyze_template 등
│   ├── templates/              # base/, report/, tables/
│   ├── assets/                 # 시각 기준 .hwpx 샘플
│   └── references/             # OWPML 포맷 참조 문서
├── README.md
└── LICENSE
```

자세한 사용법, XML 작성 가이드, 스타일 ID 맵은 [`skills/hwpx/SKILL.md`](skills/hwpx/SKILL.md)를 참조하세요.

## 라이선스

[MIT](LICENSE)
