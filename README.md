# gitto

**잘못된 깃 계정으로 커밋, 다시는 없게.** — macOS 메뉴바 앱

회사 계정과 개인 계정을 한 맥에서 쓰다 보면 언젠가 엉뚱한 이름·이메일로 커밋하거나, 엉뚱한 GitHub 계정으로 push하게 됩니다. gitto는 "계정 전환기"가 아니라 **사고 방지기**입니다.

- **프로필 전환** — 메뉴바에서 깃 신원(name/email/SSH 키/gh 계정)을 한 세트로 전환
- **레포별 고정(핀)** — 레포마다 프로필을 박아 두면, 액티브 프로필과 무관하게 항상 그 신원으로 커밋
- **불일치 감지 + 가드 훅** — 실효 신원이 어긋나면 메뉴바 배지·알림·원클릭 수정, 그래도 어긋나면 `pre-commit` 훅이 커밋을 차단
- **`.gitto` 정책 파일** — `.nvmrc`처럼 레포에 "이 레포는 어떤 계정이어야 하는가"를 선언해 팀원 모두에게 적용

## 설치

- **요구 사항**: macOS 14 Sonoma 이상
- [Releases](https://github.com/NariP/gitto-releases/releases)에서 `gitto-<버전>.dmg`를 받아 열고, `gitto.app`을 `Applications`로 드래그합니다.
- 앱은 Developer ID로 서명·공증되어 있어 그대로 실행됩니다. 만약 직접 빌드한 앱처럼 "확인되지 않은 개발자" 경고가 뜨면 **우클릭 → 열기** 또는 시스템 설정 → 개인정보 보호 및 보안 → "그래도 열기"로 한 번만 허용하면 됩니다.
- gitto는 메뉴바 전용 앱입니다(Dock 아이콘 없음). 처음 실행하면 온보딩이 두 단계를 안내합니다: **프로필 추가**(`~/.gitconfig`·`~/.ssh`에서 발견한 신원을 가져오거나 직접 입력) → **레포 폴더 지정**(예: `~/dev`).

## 핵심 기능

### 프로필 전환

프로필은 `user.name` + `user.email` + (선택) SSH 키 경로 + (선택) gh 계정 한 묶음입니다. 메뉴바에서 프로필을 고르면 gitto는

1. `~/Library/Application Support/gitto/active.gitconfig`를 그 프로필의 신원으로 다시 쓰고
2. `~/.gitconfig`에 아래 **include 한 줄만** 있는지 확인합니다(없으면 추가).

```ini
[include]
	path = ~/Library/Application Support/gitto/active.gitconfig
```

사용자의 `~/.gitconfig` 나머지는 바이트 단위로 보존됩니다. SSH 키가 있는 프로필은 `core.sshCommand = ssh -i <키> -o IdentitiesOnly=yes`도 같은 파일에 씁니다.

### 레포 고정(핀)

Repositories 탭에서 레포에 프로필을 지정하면 그 레포의 `.git/config`에 다음을 씁니다.

| 키 | 언제 |
|---|---|
| `user.name`, `user.email` | 항상 |
| `core.sshCommand` | 프로필에 SSH 키가 있을 때 |
| `gitto.ghUser` + `[credential "https://github.com"] helper` 짝 | 프로필에 gh 계정이 연결됐을 때 |

로컬 config는 글로벌보다 우선하므로, 어떤 프로필이 액티브든 이 레포는 항상 지정한 신원으로 커밋합니다. 표준 git 메커니즘이라 **gitto가 꺼져 있어도 유효**합니다. 그 외 줄(`[remote]`, `[branch]`, 직접 쓴 credential helper·`core.sshCommand`)은 건드리지 않습니다. 핀 해제 시에는 `user.name`/`user.email`·`gitto.ghUser`는 항상 지우고, `core.sshCommand`와 credential helper는 gitto가 쓴 모양일 때만 지웁니다(직접 쓴 값은 보존).

### 불일치 감지와 가드 훅

- 등록한 루트 폴더를 감시(FSEvents)해 **새로 클론된 레포를 즉시 감지**하고 "어느 프로필?" 알림을 띄웁니다.
- 각 레포의 실효 신원(로컬 > 글로벌 > include 우선순위)을 계산해 핀·`.gitto` 정책과 다르면 **메뉴바 아이콘 배지 + 알림 + 원클릭 수정**을 제공합니다.
- 핀 메뉴에서 **가드 훅**을 켜면 `.git/hooks/pre-commit`에 gitto 마커가 붙은 작은 `sh` 스크립트를 설치합니다. 커밋 시점에 실효 `user.name/email`이 핀과 다르면 커밋을 중단합니다. 이미 다른 `pre-commit`이 있으면 덮어쓰지 않고 설치를 거부합니다. 핀을 해제하면 훅도 함께 제거됩니다("핀 없으면 가드 없음").

### gh 계정 정합

프로필 편집기에서 `gh auth login`으로 로그인해 둔 GitHub 계정을 프로필에 연결할 수 있습니다.

- **프로필 전환** 시 `gh auth switch --user <계정>`을 실행해 gh 활성 계정도 함께 바뀝니다. (`~/.config/gh/hosts.yml`은 읽기만 하고 직접 쓰지 않습니다.)
- **핀된 레포**에서는 `.git/config`의 credential helper가 `gh auth token --user <계정>`으로 그 계정의 토큰을 **실행 시점에** 얻어 HTTPS push에 씁니다. 파일에는 계정명만 남고 토큰은 저장되지 않습니다. 계정이 gh에 로그인돼 있지 않으면 helper가 조용히 실패해 git이 프롬프트를 띄웁니다 — 잘못된 계정으로 push되는 일은 없습니다(safe-by-failure).

### 셸 연동(선택)

Profiles 탭 툴바의 **Set up shell integration**을 누르면 `~/Library/Application Support/gitto/shell.sh`를 쓰고, 셸 시작 파일에 마커 블록 한 개를 추가합니다(실제 파일에는 `~` 대신 절대 경로가 들어갑니다).

```sh
# >>> gitto >>>
[ -f "<홈>/Library/Application Support/gitto/shell.sh" ] && source "<홈>/Library/Application Support/gitto/shell.sh"
# <<< gitto <<<
```

`shell.sh`는 `gh` 래퍼 함수입니다. 핀된 레포 안에서 `gh`를 실행하면 핀 계정의 토큰을 주입해 **그 계정으로** 동작하고, 핀이 없는 곳에서는 평소의 `gh`와 똑같습니다. `gh auth …` 하위 명령은 주입 없이 그대로 통과합니다. zsh는 `~/.zshrc`, bash는 `~/.bash_profile`을 씁니다. 다른 셸(fish 등)은 `shell.sh`만 써 주고 `source` 줄은 직접 추가합니다.

### `.gitto` 정책 파일

레포 루트에 커밋하는 선언 파일입니다. git config 문법을 그대로 쓰며, **개인정보(실명·이메일 전체)는 넣지 않습니다** — 패턴만.

```ini
# .gitto — 이 레포의 깃 신원 정책
[identity]
	emailPattern = *@mycompany.com  ; 허용되는 커밋 이메일 glob 패턴
	profileHint = work              ; (선택) 로컬 프로필 이름 힌트
```

- 레포를 발견했을 때 `.gitto`가 있으면 `emailPattern`에 맞는 로컬 프로필을 찾습니다. 하나만 맞으면 **자동 고정 + 알림**, 여럿이거나 없으면 선택을 유도합니다.
- 불일치 감지와 가드 훅도 같은 정책을 기준으로 검증합니다.
- Repositories 탭의 핀 메뉴에서 현재 프로필로부터 `.gitto`를 **내보낼** 수 있습니다.
- `.gitto`는 클론해 온 **비신뢰 입력**으로 취급합니다. 내용은 로컬 프로필 매칭에만 쓰이며, 명령 실행·경로 쓰기·프로필 자동 생성으로 이어지지 않습니다.

## 요구 사항

| 항목 | 내용 |
|---|---|
| macOS | 14 Sonoma 이상 |
| git | 시스템 git (Xcode Command Line Tools 등). gitto 자체는 git config 파일을 직접 읽고 씁니다 |
| gh CLI (선택) | gh 계정 정합·셸 연동에만 필요. `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, 그다음 `PATH`에서 찾습니다. 계정마다 `gh auth login`으로 로그인해 두세요(다중 계정 `gh auth switch --user`를 지원하는 gh 2.40 이상 권장) |
| 셸 | 셸 연동은 zsh·bash만 자동 설정. 그 외는 수동 `source` |

## 알려진 한계

- **GitHub Enterprise 전용 레포**: gitto가 주입하는 토큰은 github.com용이라 GHE 호스트에서는 인증에 실패합니다.
- **PATH가 짧은 GUI git 클라이언트**: credential helper가 `gh`를 못 찾으면 인증에 실패하고 git이 프롬프트를 띄웁니다(잘못된 계정으로 push되지는 않음).
- **`git commit --no-verify`**(`-n`)는 모든 `pre-commit` 훅을 건너뛰므로 가드 훅도 우회됩니다. git의 설계이며, 의도적인 한 번의 탈출구로 둡니다.
- **`core.hooksPath`**로 훅 디렉터리를 바꿔 둔 레포에서는 gitto가 `.git/hooks/pre-commit`에만 쓰므로 가드 훅이 실행되지 않습니다.
- 핀은 github.com용 **레포 로컬 credential helper를 gitto 것으로 리셋**합니다(직접 설정한 helper는 핀 해제 시 보존됩니다).
- gitto는 SSH 키를 생성하지 않고, 개인키를 읽지 않습니다(`~/.ssh/config`와 `.pub` 주석만 읽습니다).

## gitto가 건드리는 파일

| 위치 | 내용 |
|---|---|
| `~/.gitconfig` | `[include] path = ~/Library/Application Support/gitto/active.gitconfig` 한 줄 (실제로는 절대 경로로 기록) |
| `~/Library/Application Support/gitto/` | `profiles.json`, `repos.json`, `active.gitconfig`, `shell.sh` |
| 핀된 레포의 `.git/config` | `user.name`, `user.email`, (SSH 키 프로필) `core.sshCommand`, (gh 프로필) `gitto.ghUser` + github.com credential helper 짝 |
| 핀된 레포의 `.git/hooks/pre-commit` | 가드 훅을 켠 경우에만. `# gitto-guardhook v1` 마커로 시작 |
| `~/.zshrc` 또는 `~/.bash_profile` | 셸 연동을 켠 경우에만. `# >>> gitto >>>` … `# <<< gitto <<<` 블록 |
| gh | `gh auth switch --user`를 실행할 뿐, gh의 설정 파일을 직접 쓰지 않음 |

UserDefaults·키체인·로그인 항목(LaunchAgent)은 사용하지 않습니다.

## 제거(Uninstall)

### 앱에서 한 번에 되돌리기

Profiles 탭 툴바 오른쪽 **⋯** 메뉴 → **Undo all gitto changes…** → 확인. 다음 순서로 되돌립니다.

1. 핀된 모든 레포의 핀 해제(가드 훅도 함께 제거)
2. 셸 연동 제거(rc 파일의 마커 블록 + `shell.sh`)
3. `~/.gitconfig`의 include 줄 제거(+ 액티브 프로필 해제)

한 단계가 실패해도 나머지는 계속 진행하고, 실패한 항목을 안내 문구에 표시합니다. 완료되면 "이제 앱을 지워도 된다"는 안내와 함께 남아 있는 데이터 폴더 경로를 알려 줍니다. 프로필과 등록 폴더는 그 폴더에 남아 있으니, 돌아올 생각이 없다면 아래 5번을 따라 지우세요.

### 수동으로 제거하기

1. gitto를 종료하고 `/Applications/gitto.app`을 휴지통으로.
2. `~/.gitconfig`에서 `[include]` 아래 `path = …/gitto/active.gitconfig` 줄을 지웁니다(`[include]` 헤더에 다른 줄이 없으면 헤더도).
3. 핀했던 레포마다 `.git/config`에서 `user.name`, `user.email`, gitto 모양(`ssh -i … -o IdentitiesOnly=yes`)의 `core.sshCommand`, `gitto.ghUser`, `[credential "https://github.com"]`의 gitto helper 짝(`helper =` 빈 줄 + `!f() { t=$(gh auth token --user …` 줄)을 지웁니다. `.git/hooks/pre-commit`이 `# gitto-guardhook`로 시작하면 그 파일을 삭제합니다.
4. `~/.zshrc`(zsh) 또는 `~/.bash_profile`(bash)에서 `# >>> gitto >>>`부터 `# <<< gitto <<<`까지 블록을 지웁니다.
5. `~/Library/Application Support/gitto`를 삭제합니다. 예전 샌드박스 빌드를 썼다면 `~/Library/Containers/com.narip.gitto`도 남아 있을 수 있으니 함께 삭제합니다.
6. gitto가 gh 활성 계정을 바꿔 두었다면 `gh auth switch`로 원하는 계정으로 돌아갑니다.
7. 알림 센터의 gitto 항목은 앱을 지우면 함께 사라집니다.

## 라이선스

gitto는 [PolyForm Noncommercial 1.0.0](LICENSE) 조건으로 배포됩니다. 개인·교육·비영리 목적의 사용은 자유이며, 상업적 사용에는 저작권자의 허락이 필요합니다.

이 레포는 **배포 전용**입니다. 소스 코드는 비공개이며, 버그 제보와 제안은 이 레포의 [Issues](https://github.com/NariP/gitto-releases/issues)로 남겨 주세요.
