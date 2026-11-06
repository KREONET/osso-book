# 부록 G. 패키지 관리자 — 설치, 목록 기록, 재설치 {#appendix-g}

**멘탈모델**(머릿속에 그려 두는 작동 모형): 패키지 관리자는 설치를 명령 한 줄로 줄여 줄 뿐 아니라, **내가 무엇을 설치했는지를 파일 하나로** 남겨 준다. 그 파일이 있으면 새 컴퓨터를 몇 분 만에 내 컴퓨터와 같은 상태로 만들 수 있다.

## G.1 패키지 관리자가 하는 일 {#g-1-what}

웹사이트에서 `setup.exe`를 내려받아 "다음 → 다음 → 완료"를 누르는 방식과 비교하면 이렇습니다.

| | 직접 내려받기 | 패키지 관리자 |
| --- | --- | --- |
| 어디서 받나 | 검색 결과의 아무 사이트 (가짜 사이트 위험) | 정해진 저장소 한 곳 |
| 진짜인가 | 사람이 판단 | 서명·해시를 자동 검증 |
| 필요한 다른 것들 | 설치하다 "OO가 없습니다"로 알게 됨 | 의존성을 자동으로 함께 설치 |
| 업데이트 | 프로그램마다 따로 | 한 줄로 전부 (`apt upgrade`, `brew upgrade`, `winget upgrade --all`) |
| 지우기 | 찌꺼기가 남기 쉬움 | 설치한 파일을 알고 있어 깨끗이 |
| 무엇을 설치했나 | 기억에 의존 | **목록으로 내보내고, 그 목록으로 다시 설치** |

이 책에서 가장 중요한 것은 마지막 줄입니다. [Part 1](01-image.md)이 "서버를 파일(이미지)로", [§1.3](01-image.md)의 점 파일이 "설정을 파일로"였다면, 패키지 목록은 **"설치한 프로그램을 파일로"** 만드는 일입니다.

## G.2 운영체제별 패키지 관리자 {#g-2-os}

| 환경 | 패키지 관리자 | 목록 내보내기 | 목록으로 다시 설치 |
| --- | --- | --- | --- |
| Debian · Ubuntu | `apt` | `apt-mark showmanual > pkgs.txt` | `xargs -a pkgs.txt sudo apt-get install -y` |
| RHEL · Rocky | `dnf` | `dnf repoquery --userinstalled --qf '%{name}' -q > pkgs.txt` | `xargs -a pkgs.txt sudo dnf install -y` |
| macOS (리눅스에서도 동작) | **Homebrew** (`brew`) | `brew bundle dump` → `Brewfile` | `brew bundle` |
| Windows 10·11 | **WinGet** (`winget`) | `winget export -o apps.json` | `winget import -i apps.json` |
| 파이썬 · conda | `pip`/`uv`, conda | lock 파일, `environment.yml` | → [§1.8](01-image.md) |

Windows에는 WinGet 말고도 커뮤니티의 Chocolatey·Scoop이 있고, 리눅스에는 배포판과 상관없이 쓰는 Flatpak·Snap도 있습니다. 원리는 모두 같습니다.

### 언어별 패키지 관리자 — 버전·가상환경·패키지 {#g-2-languages}

OS 패키지 관리자 위에 **프로그래밍 언어마다** 관리 도구가 한 층 더 있습니다. 하는 일은 세 가지로 나뉩니다.

- **버전 관리:** 같은 언어의 여러 버전을 나란히 설치하고 골라 쓴다 (파이썬 3.11과 3.12)
- **가상환경:** 프로젝트마다 독립된 패키지 공간을 주어 프로젝트끼리 충돌하지 않게 한다
- **패키지 관리:** 그 언어의 라이브러리를 설치하고 버전을 고정한다

| 언어 | 버전 관리 | 가상환경 | 패키지 관리 |
| --- | --- | --- | --- |
| Python | pyenv | venv, virtualenv | pip (+ poetry, pipenv, [uv](https://docs.astral.sh/uv/)) |
| Node.js | nvm, n, fnm | — (프로젝트의 `node_modules`) | npm, yarn, pnpm |
| Ruby | rbenv, rvm | bundler | gem |
| Go | gvm, goenv | — (`go.mod`가 프로젝트 단위) | go mod |
| Rust | rustup | rustup override | cargo |
| Java · Kotlin · Scala | sdkman, jenv | — | maven, gradle, sbt |
| C# (.NET) | dotnet-install | `global.json` | NuGet |
| C · C++ | — | — | OS 패키지 관리자, Conan, vcpkg |

그리고 이 표 전체를 가로지르는 것이 **conda**입니다. 파이썬뿐 아니라 R·C 라이브러리·CUDA처럼 언어 밖의 의존성까지 한 환경에 묶어서, 데이터 과학·AI 연구에서 많이 씁니다. 라이선스 함정(anaconda 기본 채널)과 대안(miniforge, 또는 miniconda에 `conda-forge` 채널 등록)은 [§1.8](01-image.md)을 보세요.

```bash
# miniconda를 이미 쓰고 있다면 — 커뮤니티 채널을 우선으로
conda config --add channels conda-forge
conda config --set channel_priority strict
```

> `apt-mark showmanual`은 **내가 직접 설치한 것**만 보여 줍니다(의존성으로 딸려 온 것은 빠짐). 갓 만든 Ubuntu 24.04 컨테이너에서도 91개가 나오는데, 배포판이 처음부터 "직접 설치한 것"으로 표시해 둔 기본 패키지들입니다. 다른 서버에 옮길 때는 내가 추가한 것만 남기도록 한 번 훑어보세요. `dnf repoquery --userinstalled`도 같습니다.

## G.3 Homebrew와 Brewfile {#g-3-homebrew}

[Homebrew](https://brew.sh/)는 macOS의 사실상 표준 패키지 관리자이고 리눅스·WSL에서도 동작합니다. 설치 목록을 **`Brewfile`** 이라는 텍스트 파일로 다룹니다.

```bash
# 지금 깔린 것을 Brewfile로 — --describe는 줄마다 설명 주석을 달아 준다
brew bundle dump --describe --file=Brewfile_$(date +%Y%m%d)
# dotfiles 저장소에 둘 거라면 이름을 고정하고 덮어쓰기
brew bundle dump --describe --file=~/dotfiles/Brewfile --force
# 새 맥에서: Brewfile대로 설치
brew bundle --file=~/dotfiles/Brewfile
# 빠진 게 있는지 확인만
brew bundle check --file=~/dotfiles/Brewfile
```

Brewfile은 이렇게 생겼습니다. 한 줄이 프로그램 하나입니다.

```ruby
# 명령줄 도구
brew "git"
brew "htop"
brew "postgresql@16", restart_service: true
# 그래픽 앱 (macOS)
cask "firefox"
cask "visual-studio-code"
# 앱스토어 앱 (macOS, id로 지정)
mas "Refined GitHub", id: 1519867270
# VS Code 확장
vscode "editorconfig.editorconfig"
```

`brew`는 명령줄 도구, `cask`는 그래픽 앱, `mas`는 맥 앱스토어 앱, `vscode`는 편집기 확장입니다. 앱스토어 앱까지 목록에 들어가려면 앱스토어용 명령줄 도구 `mas`가 설치되어 있어야 합니다(`brew install mas`). `cask`·`mas`는 macOS 전용이라 리눅스에서는 건너뛰게 씁니다(`brew "gnupg" if OS.mac?` 같은 조건). `brew bundle cleanup --force`는 반대로 **Brewfile에 없는 것을 지웁니다**. 목록을 기준으로 삼는 선언형 운영(원하는 최종 상태를 적어 두면 도구가 그 상태로 맞추는 방식)입니다([Part 4 compose](04-container.md)와 같은 생각).

## G.4 WinGet — 윈도우의 패키지 관리자 {#g-4-winget}

**WinGet**(Windows Package Manager)은 맥·리눅스가 패키지 관리자 덕분에 편리해진 것을 보고 마이크로소프트가 공식으로 만든 명령줄 패키지 관리자입니다. Windows 10 이상에서 쓸 수 있고, 최신 Windows에는 *앱 설치 관리자*(App Installer)와 함께 이미 들어 있습니다. 없으면 Microsoft Store에서 [앱 설치 관리자](https://apps.microsoft.com/detail/9nblggh4nns1)를 설치합니다. PowerShell이나 명령 프롬프트에서 씁니다.

```powershell
# 찾기 — 패키지 ID를 확인한다
winget search vscode
# 설치 — ID로 정확히
winget install --id Microsoft.VisualStudioCode -e
# 여러 개를 한 번에
winget install Microsoft.PowerToys Google.Chrome
# 깔린 것 전부 업데이트
winget upgrade --all
# 지금 PC의 앱 목록을 JSON으로 (버전까지 고정하려면 --include-versions)
winget export -o apps.json
# 새 PC에서: 목록대로 설치
winget import -i apps.json
```

설치 명령을 직접 찾기 번거로우면 [winstall.app](https://winstall.app/)에서 앱을 고르면 `winget install` 명령을 만들어 줍니다. 패키지는 기본으로 마이크로소프트의 공개 저장소(`winget source list`로 확인)에서 오고, 기관 내부 저장소를 `winget source add`로 더할 수도 있습니다.

`export`는 *프로그램 추가/제거*에 있는 앱 중 WinGet 저장소에서 찾을 수 있는 것만 목록에 넣고, 못 찾은 것은 경고로 알려 줍니다. 만들어진 JSON은 직접 고쳐서 필요 없는 앱을 지워도 됩니다. 팀원에게 나눠 주면 **모두 같은 도구 세트**로 시작합니다.

⚠️ `--accept-package-agreements`는 각 프로그램의 **라이선스 동의를 대신 눌러 주는** 옵션입니다. 편하지만, 무엇에 동의하는지 모르고 누르게 됩니다. 기관 PC에 일괄 설치할 때는 목록의 라이선스를 한 번 확인하세요([부록 B](B-licenses.md)).

## G.5 dotfiles 저장소에 함께 보관하기 — clone 한 번으로 새 컴퓨터 설정 {#g-5-dotfiles}

설정(점 파일)과 설치 목록을 **한 저장소**에 두면, 새 컴퓨터를 내 컴퓨터로 만드는 일이 이렇게 줄어듭니다.

```
dotfiles/
├── .zshrc  .vimrc  .gitconfig
├── .ssh/conf.d/            ← 팀 공용 접속 별명 (개인 키는 절대 X)
├── Brewfile                ← macOS
├── apps.json               ← Windows (winget export)
├── pkgs-apt.txt            ← Ubuntu (apt-mark showmanual)
└── install.sh              ← 링크 걸고, 목록대로 설치하는 스크립트
```

```bash
git clone https://github.com/⟨myid⟩/dotfiles.git ~/dotfiles
~/dotfiles/install.sh
```

다른 사람들의 사례를 보고 싶으면 GitHub의 **`dotfiles` 토픽**을 보면 됩니다. 저장소에 붙은 주제 태그로, 수많은 사람의 dotfiles 저장소가 별 개수 순으로 모여 있습니다.

- GitHub 토픽: <https://github.com/topics/dotfiles> — 같은 식으로 `https://github.com/topics/⟨주제⟩`를 바꿔 넣으면 그 주제의 저장소가 모인다
- dotfiles 안내 모음: <https://dotfiles.github.io/> — 관리 도구(GNU Stow, chezmoi, yadm 등)와 예시

⚠️ 공개 저장소에 올리기 전에 **비밀이 없는지** 꼭 확인합니다. 점 파일에는 토큰·사내 호스트명·IP가 섞여 들어가기 쉽습니다. 개인 키와 토큰 파일은 `.gitignore`로 막고, 사내 접속 정보는 공개 저장소가 아니라 팀 내부 저장소에 둡니다([§1.3](01-image.md)).

## G.6 안전하게 쓰기 {#g-6-safety}

- **저장소가 곧 신뢰다.** 패키지 관리자는 저장소의 서명을 검증할 뿐, 저장소 자체를 판단하지는 않습니다. 공식 저장소 밖(Ubuntu PPA, Homebrew tap, 개별 회사 저장소)을 추가하는 것은 **그 운영자를 믿겠다**는 결정입니다.
- **나가는 길과 짝이다.** 서버가 패키지를 받는 곳이 정해져 있으면 egress 허용 목록도 짧아집니다. [실습⑥](labs/lab-06.md)의 squid가 apt·yum·docker 저장소만 열어 두는 이유입니다.
- **라이선스도 패키지에 딸려 온다.** 같은 `conda` 명령이라도 채널에 따라 라이선스가 다릅니다(anaconda ▶ miniforge, [§1.8](01-image.md)).
- **업데이트를 미루지 않는다.** 한 줄로 전부 업데이트할 수 있다는 점이 패키지 관리자의 가장 큰 보안 이득입니다. 직접 받은 프로그램은 대개 업데이트되지 않은 채 남습니다.

더 깊이: [Homebrew Bundle](https://docs.brew.sh/Brew-Bundle-and-Brewfile), [WinGet 문서](https://learn.microsoft.com/en-us/windows/package-manager/winget/), [winget export](https://learn.microsoft.com/en-us/windows/package-manager/winget/export), [winget import](https://learn.microsoft.com/en-us/windows/package-manager/winget/import), [apt-mark(8)](https://manpages.debian.org/apt-mark), [GitHub dotfiles 토픽](https://github.com/topics/dotfiles), [dotfiles.github.io](https://dotfiles.github.io/).
