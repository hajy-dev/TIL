# Git 설정과 GitHub 연동

> 작성일: 2026-04-25  
> 환경: macOS  
> 배경: 맥북 초기 세팅 중 Git 환경 구축 및 GitHub 계정 연동

## TL;DR

- 기본 설정 외에 macOS는 `core.autocrlf=input`, `core.ignorecase=false` 추가 필수
- SSH 키 생성 시 이메일은 반드시 `-C` 플래그로 전달
- GitHub Username은 마침표(`.`) 사용 불가 — 영문자/숫자/단일 하이픈만
- 다중 계정은 한 기기에 합치지 말고 **기기-계정 1:1 매핑**으로 분리

---

## 1. Git 설치

macOS는 Apple이 번들한 Git이 기본 포함되어 있다.

\`\`\`bash
git --version
# git version 2.x.x (Apple Git-xxx)
\`\`\`

이번엔 공식 사이트에서 직접 다운로드하여 설치했다. 
Homebrew(`brew install git`)로 설치하는 방법도 있으나 이 부분은 별도로 정리할 예정.

---

## 2. 기본 설정

### 2.1 사용자 정보 등록

커밋 작성자로 기록될 정보를 등록한다.

\`\`\`bash
git config --global user.name "ha.jy"
git config --global user.email "ha.jydev@gmail.com"
\`\`\`

> 이메일은 **GitHub 계정에 등록된 이메일**과 일치해야 잔디(contribution graph)에 반영된다.

### 2.2 기본 브랜치를 main으로

\`\`\`bash
git config --global init.defaultBranch main
\`\`\`

과거 Git은 `master`를 기본 브랜치로 사용했지만 GitHub/GitLab 모두 `main`으로 통일됐다.

---

## 3. macOS 전용 옵션 (협업 고려)

기본 설정만으로도 Git은 동작하지만 **팀 협업**을 고려한다면 옵션을 추가해야 한다.

### 3.1 core.autocrlf=input

\`\`\`bash
git config --global core.autocrlf input
\`\`\`

**왜 필요한가:**

- 줄바꿈 문자가 OS마다 다름
  - Windows: `CRLF` (`\r\n`)
  - macOS/Linux: `LF` (`\n`)
- 같은 파일이라도 OS에 따라 줄바꿈이 달라 Git이 "변경됨"으로 판단
- Windows 팀원과 협업 시 의미 없는 대규모 diff 발생 → 코드 리뷰 불가능

**옵션 값별 동작:**

| 값 | push 시 | clone/pull 시 | 권장 OS |
|----|---------|---------------|---------|
| `true` | CRLF → LF | LF → CRLF | Windows |
| `input` | CRLF → LF | 변환 없음 | **macOS, Linux** |
| `false` | 변환 없음 | 변환 없음 | (권장 안 함) |

저장소에는 항상 LF로만 저장하는 것이 표준. macOS는 로컬도 LF를 쓰므로 `input`이 적합.

### 3.2 core.ignorecase=false

\`\`\`bash
git config --global core.ignorecase false
\`\`\`

**왜 필요한가:**

- macOS APFS 파일시스템은 기본적으로 **대소문자 구분 안 함**
- `UserController.java`와 `usercontroller.java`가 같은 파일로 취급됨
- 하지만 **Linux 서버는 구분함**

**실제 위험 시나리오:**

\`\`\`bash
# macOS에서 파일명 변경
mv UserController.java usercontroller.java
git status
# → 변경사항 없음 (macOS는 같은 파일로 인식)
\`\`\`

이 상태로 push → Linux 배포 서버에서 `ClassNotFoundException` 발생.

`false`로 두면 Git이 파일시스템과 무관하게 항상 대소문자를 구분한다.

### 3.3 설정 확인

\`\`\`bash
git config --global --list
\`\`\`

---

## 4. SSH 키 생성

### 4.1 ⚠️ 겪은 에러: Too many arguments

처음 명령어를 이렇게 입력했다.

\`\`\`bash
ssh-keygen -t ed25519 "ha.jydev@gmail.com"
# Too many arguments.
\`\`\`

`ssh-keygen` 입장에서 따옴표로 감싼 이메일이 어떤 옵션인지 인식하지 못해 발생한 에러였다.

### 4.2 해결: -C 플래그 사용

이메일 앞에 `-C` 플래그를 붙여 "주석(comment)"임을 명시해야 한다.

\`\`\`bash
ssh-keygen -t ed25519 -C "ha.jydev@gmail.com"
\`\`\`

| 플래그 | 의미 |
|--------|------|
| `-t ed25519` | 키 타입 (ed25519는 현재 권장 알고리즘) |
| `-C "이메일"` | 키에 식별용 주석 추가 |

이메일은 인증에 쓰이지 않고, GitHub에서 키를 식별하는 라벨일 뿐이다.

### 4.3 생성된 파일

\`\`\`
~/.ssh/id_ed25519       # 개인키 (절대 공유 금지)
~/.ssh/id_ed25519.pub   # 공개키 (GitHub에 등록)
\`\`\`

---

## 5. GitHub에 공개키 등록

### 5.1 클립보드에 복사

\`\`\`bash
pbcopy < ~/.ssh/id_ed25519.pub
\`\`\`

`pbcopy`는 macOS 전용. Windows의 `clip <`에 해당.

### 5.2 GitHub 등록

1. GitHub → **Settings** → **SSH and GPG keys** → **New SSH key**
2. Title: 기기 식별용 이름 (예: `MacBook`)
3. Key: 클립보드 내용 붙여넣기
4. **Add SSH key**

### 5.3 연결 테스트

\`\`\`bash
ssh -T git@github.com
\`\`\`

처음 접속 시 호스트 신뢰 여부를 묻는다 → `yes`

성공 메시지:

\`\`\`
Hi [Username]! You've successfully authenticated, but GitHub does not provide shell access.
\`\`\`

`does not provide shell access`는 정상 메시지. GitHub은 서버 쉘 접속을 허용하지 않지만 Git 작업용 인증은 통과됐다는 뜻이다.

---

## 6. GitHub Username 변경

기본 Username `Ha-Jaeyoung`에서 `hajy-dev`로 변경했다.

### 6.1 명명 규칙

GitHub Username은 다음 규칙을 따른다.

> Username may only contain alphanumeric characters or single hyphens, and cannot begin or end with a hyphen.

- 영문자(a-z, A-Z)와 숫자(0-9)만
- 하이픈(`-`)은 단일로만, 시작/끝 불가
- **마침표(`.`), 언더스코어(`_`) 사용 불가**

### 6.2 변경 시 주의

- 변경하면 구 Username을 다른 사람이 가져갈 수 있음
- 기존 리포 URL이 모두 바뀜 (자동 리다이렉트 제공되지만 영구 보장 X)
- SSH 인증은 영향 없음 (계정 자체는 동일)

리포가 거의 없는 초기 단계에 변경하는 것이 안전하다.

### 6.3 후보 결정 과정

이메일 ID `ha.jydev`를 그대로 쓸 수 없어(마침표 불가) 후보를 추렸다.

| 후보 | 의미 분리 | 비고 |
|------|----------|------|
| `hajydev` | 분리 없음 | 이메일과 거의 동일 |
| `ha-jydev` | `ha` + `jydev` | 이메일 일관성 ↑ |
| `hajy-dev` | `hajy` + `dev` | 직무 정체성 ↑ |

직무 정체성을 명시할 수 있는 `hajy-dev`로 결정.

---

## 7. 다중 계정 운영 전략

`ha.jydev`(개인 계정)와 `rhu2222`(팀 프로젝트 계정)를 모두 사용 중이지만, **한 기기에 두 계정을 모두 연결하지 않기로** 결정했다.

### 7.1 기기-계정 1:1 매핑

| 기기 | 계정 | 용도 |
|------|------|------|
| MacBook | hajy-dev | 개인 TIL, 사이드 프로젝트, 포트폴리오 |
| 학원 노트북 (AnyDesk 원격) | rhu2222 | AI-Pass 팀 프로젝트 |

### 7.2 다중 연결을 안 한 이유

1. **SSH 분기 설정 부담** — 계정별 키 생성 + `~/.ssh/config` 호스트 분기 + 리포별 user.name/email 설정
2. **실수 시 복구 어려움** — 잘못된 계정으로 커밋하면 git 히스토리 수정 매우 까다로움
3. **물리적 분리가 가장 안전** — 기기가 다르면 계정 혼동이 원천 차단됨

### 7.3 프로젝트 완료 후 처리

AI-Pass 종료 후 `hajy-dev` 계정으로 Fork하여 개인 포트폴리오로 보존할 계획.

---

## 8. 회고

### 잘 된 것

- macOS 전용 옵션을 처음부터 협업·배포 환경 고려해서 추가
- Username 변경을 리포가 거의 없는 초기 시점에 진행 (안전한 타이밍)
- 다중 계정을 무리하게 한 기기에 합치지 않고 단순한 분리 전략 채택

### 막혔던 것

- `ssh-keygen`에서 `-C` 플래그 누락 → "Too many arguments" 에러
  - 에러 메시지가 명확해서 빠르게 해결
- GitHub Username 명명 규칙(마침표 불가)을 모르고 시도 → 실패 후 `hajy-dev`로 결정

---

## 관련 글

- [macOS 초기 개발환경 세팅](../macOS/2026-04-25-mac-initial-setup.md)