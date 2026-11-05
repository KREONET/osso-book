# Part 4. 컨테이너 운영

> 담당 랩 [실습⑨](labs/lab-09.md) · [실습⑨′](labs/lab-09p.md) + 손놀림.

이 장의 한 줄: **컨테이너는 "작은 VM"이 아니라 "선언된 프로세스"다.** 선언은 파일에 남고, 나갈 수 있는 길도 선언으로 그린다.

## 4.0 컨테이너 이전에 — 프로세스, 환경변수, 격리

**멘탈모델.** 컨테이너는 새 발명이 아니다. **운영체제가 원래 하던 "나눠 쓰기"를 더 촘촘하게 쓴 프로세스**다. 그래서 컨테이너를 이해하려면 먼저 프로세스가 어떻게 태어나고, 무엇을 물려받는지를 알아야 한다.

### 컴퓨터는 처음부터 나눠 쓰는 물건이었다

컴퓨터는 한 사람의 책상이 아니라 **중앙의 메인프레임**에서 시작했습니다. 비싼 기계 한 대를 여러 사람이 단말기로 붙어 시간을 쪼개 썼습니다(시분할). 그러니 운영체제는 처음부터 이런 일을 해야 했습니다.

- **사용자별 격리:** 내 프로세스와 파일은 남이 못 건드린다 (UID, 파일 권한)
- **자원 쿼터:** 한 사람이 디스크·CPU·메모리를 다 차지하지 못하게 한다 (디스크 쿼터, `ulimit`)
- **보이는 범위 제한:** 1979년 유닉스 V7의 `chroot`는 "이 프로세스에게는 이 디렉터리가 `/`다"라고 속였다

리눅스는 이걸 두 갈래로 더 촘촘하게 만들었습니다.

| 기능 | 하는 일 | 나누는 것 | 들어온 때 |
| --- | --- | --- | --- |
| **namespace** | **무엇이 보이나** (격리) | 프로세스 번호(pid), 네트워크(net), 마운트(mnt), 호스트명(uts), 사용자(user) 등 | 2002년 mnt부터 하나씩, user는 2013년 |
| **cgroup** | **얼마나 쓰나** (쿼터·회계) | CPU, 메모리, 디스크 I/O, 프로세스 수 | 2008년 (2.6.24) |

cgroup은 구글 엔지니어들이 "process containers"라는 이름으로 제안한 것이 시작이고, 바로 위의 쿼터 일을 프로세스 묶음 단위로 하는 장치입니다. **컨테이너 = namespace(격리) + cgroup(쿼터) + 이미지(파일시스템 꾸러미)** 를 한 번에 쓰기 좋게 포장한 것입니다. LXC(2008)가 먼저 했고, Docker(2013)가 이걸 "이미지 한 줄로 배포"하는 경험으로 만들었습니다.

```bash
# 지금 내 셸이 속한 namespace들 — 이 번호가 같으면 같은 세상을 본다
ls -l /proc/$$/ns
# 내 셸이 속한 cgroup
cat /proc/self/cgroup
# 시스템 전체를 cgroup 트리로
systemd-cgls --no-pager | head -30
```

`docker run --memory 256m`은 그 컨테이너의 cgroup에 메모리 한도를 적는 것이고, 컨테이너 안에서 `ps`를 치면 프로세스가 몇 개 안 보이는 건 pid namespace 때문입니다.

### 네트워크 스택 한 벌 — 컨테이너 네트워크가 유독 어려운 이유

net namespace를 새로 만들면 **네트워크 스택이 통째로 한 벌** 생깁니다. 인터페이스, IP, 라우팅 표, 방화벽 규칙, 소켓과 포트가 모두 따로입니다.

```bash
docker run --rm alpine ip addr
```

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    inet 127.0.0.1/8 scope host lo
2: eth0@if61: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
```

호스트에 인터페이스가 여럿 있어도 컨테이너에게는 `lo`와 `eth0` 둘뿐입니다. `eth0@if61`의 `@if61`은 "호스트 쪽 61번 인터페이스와 짝인 가상 케이블(veth)"이라는 뜻입니다.

그리고 리눅스 커널에서 네트워크는 **큰 덩어리**입니다. 커널 소스(Linux 7.4-rc1, 2026-11-03)의 C·헤더·어셈블리·Rust 코드 약 3,880만 줄을 디렉터리별로 세어 보면 이렇습니다.

| 디렉터리 | 무엇 | 줄 수 | 비율 |
| --- | --- | ---: | ---: |
| `drivers/` | **장치 드라이버** 전체 | 2,677만 | **69.0%** |
| ├ `drivers/net/` | 네트워크 카드 드라이버 (유선 268만·무선 205만 포함) | 537만 | 13.8% |
| ├ `drivers/gpu/` | 그래픽 드라이버 | 872만 | 22.5% |
| │ └ 그중 `amd/include/asic_reg/` | AMD GPU 레지스터 정의 (**자동 생성된 헤더**) | 524만 | 13.5% |
| └ 나머지 | 미디어·SCSI·USB·클럭 등 | 1,268만 | 32.7% |
| `arch/` | CPU 아키텍처별 코드 (x86, ARM, RISC-V…) | 254만 | 6.5% |
| `sound/` | 사운드 | 171만 | 4.4% |
| `fs/` | 파일시스템 (ext4, XFS, NFS…) | 167만 | 4.3% |
| `net/` | **네트워크 프로토콜 스택** (TCP/IP, netfilter, 브리지…) | 130만 | 3.4% |
| `kernel/` | 프로세스·스케줄러·시그널 | 59만 | 1.5% |
| `mm/` | 메모리 관리 | 23만 | 0.6% |
| `security/` | SELinux·AppArmor 등 | 13만 | 0.3% |
| 그 밖 | `include/`, `tools/`, `lib/`, `crypto/`, `rust/` … | 387만 | 10.0% |

읽는 법은 셋입니다.

- **커널의 3분의 2 이상은 드라이버**입니다. 커널의 "본체"라 할 만한 `kernel/`(프로세스·스케줄러)과 `mm/`(메모리)은 합쳐도 2%가 안 됩니다. 커널이 하는 일의 대부분은 세상의 온갖 하드웨어와 말을 맞추는 일입니다.
- **네트워크를 합치면 약 18%**입니다. 프로토콜 스택(`net/`) + 네트워크 카드 드라이버(`drivers/net/`) + 네트워크 헤더(`include/net/`)가 680만 줄. 사람이 쓴 코드가 아닌 자동 생성 헤더(AMD GPU 레지스터 524만 줄)를 빼고 보면 **약 20%, 다섯 줄에 한 줄**이 네트워크입니다.
- 그 헤더를 빼면 **드라이버 중 가장 큰 가족은 네트워크 카드**입니다(537만 줄, GPU는 349만 줄). 프로토콜 스택만 봐도 프로세스·스케줄러·메모리 관리를 합친 것보다 큽니다.

**이 비율은 15년째 그대로입니다.** 위스콘신대의 드라이버 연구([Kadav & Swift, 「Understanding Modern Device Drivers」, ASPLOS 2012](https://pages.cs.wisc.edu/~kadav/app/papers/driverstudy.pdf))는 Linux 2.6.37.6(2011년 4월)을 분석해 "장치 드라이버가 리눅스 코드의 **70%**"이고, 드라이버 코드 중 **네트워크 드라이버가 25%**라고 보고했습니다. 위 표(2026년)도 드라이버 69%, 자동 생성 헤더를 뺀 드라이버 중 네트워크 카드 드라이버 25%(537만 / 2,153만 줄)로 거의 같습니다. 커널 전체 크기로는 문서·장치 트리 등 모든 파일을 합쳐 2025년 초에 처음으로 4천만 줄을 넘었고, 그 절반 이상이 드라이버라는 보도도 있습니다([heise, 「Linux breaks through the 40 million line mark」, 2025-01-25](https://www.heise.de/en/news/Linux-breaks-through-the-40-million-line-mark-10256571.html)). 기준(어떤 파일을 세나)에 따라 숫자는 조금씩 달라지니, 위 표는 아래 명령으로 누구나 다시 셀 수 있게 해 두었습니다.

그만큼 기능도, 규칙도, 함정도 많습니다. 컨테이너가 이 스택을 한 벌씩 따로 갖고, 그 사이를 브리지·NAT·방화벽 규칙으로 잇다 보니 [실습③](labs/lab-03.md)의 "ufw가 왜 뚫리지", [⑨′](labs/lab-09p.md)의 egress, [4.5](#45-이미지-빌드--dockerfile-비루트non-root-멀티스테이지-대역-겹침)의 대역 겹침 같은 문제가 생깁니다.

```bash
# 직접 세어 보기 — 최신 소스를 얕게 받아서 (약 40초, 디스크 2 GB)
git clone --depth 1 https://github.com/torvalds/linux.git && cd linux
# 디렉터리별 C·헤더 줄 수
for d in drivers drivers/net net fs kernel mm; do
  printf '%-12s ' "$d"
  find "$d" -name '*.[ch]' -print0 | xargs -0 cat | wc -l
done
```

### 같은 컴퓨터 안의 대화 — 파일로 여는 소켓 (unix socket)

소켓이라고 하면 보통 `IP:포트`(1~65535)를 떠올립니다. 그런데 리눅스 안에서 프로그램끼리 나누는 대화는 대부분 **포트가 아니라 파일 경로로 여는 소켓**, 곧 **유닉스 도메인 소켓**으로 이루어집니다. 네트워크 스택(TCP/IP)을 거치지 않고 커널 안에서 바로 이어지므로 빠르고, 파일이므로 **파일 권한으로 누가 붙을 수 있는지 정합니다.**

```bash
# 이 컴퓨터의 소켓 수 — 유닉스 vs TCP vs UDP
echo "unix: $(ss -xa | tail -n +2 | wc -l)  tcp: $(ss -ta | tail -n +2 | wc -l)  udp: $(ss -ua | tail -n +2 | wc -l)"
# 듣고 있는 유닉스 소켓과 그 주인 프로세스
sudo ss -xlp | head -20
```

이 책을 쓴 리눅스 PC에서는 `unix: 1009  tcp: 99  udp: 8`이었습니다. 열에 아홉이 파일 소켓입니다. 익숙한 것들이 다 여기 있습니다.

| 소켓 파일 | 누가 쓰나 |
| --- | --- |
| `/run/systemd/…`, `/run/systemd/journal/…` | `systemctl`이 systemd에게, 프로그램이 journald에게 로그를 보낼 때 |
| `/dev/log` | 전통적인 syslog 입구 (요즘은 journald로 이어지는 링크) |
| `/run/dbus/system_bus_socket` | D-Bus — 데스크톱·시스템 서비스끼리의 메시지 버스(RPC) |
| `/var/run/docker.sock` | `docker` 명령이 Docker 데몬에게 |
| `/run/containerd/containerd.sock` | Docker 데몬이 containerd에게 |
| `/var/run/postgresql/.s.PGSQL.5432` | `psql`이 같은 서버의 PostgreSQL에게 (포트 번호가 이름에 남아 있다) |
| `$SSH_AUTH_SOCK` | `ssh`가 ssh-agent에게 키 서명을 부탁할 때 |

`docker` 명령도 사실 이 파일 소켓 위에서 **HTTP API**를 부르는 클라이언트일 뿐입니다. `curl`로 똑같이 부를 수 있습니다.

```bash
# docker version 과 같은 질문을 curl로
curl -s --unix-socket /var/run/docker.sock http://localhost/version
# docker ps 와 같은 질문
curl -s --unix-socket /var/run/docker.sock http://localhost/containers/json
ls -l /var/run/docker.sock
```

```text
srw-rw---- 1 root docker 0 … /var/run/docker.sock
```

맨 앞의 `s`가 소켓 파일이라는 표시이고, `root:docker`·`rw-rw----`가 곧 **"누가 Docker를 부릴 수 있나"의 전부**입니다. 그래서:

- `docker` 그룹에 넣는 것 = 이 파일에 쓸 권한을 주는 것 = 사실상 root를 주는 것 (4.3).
- 컨테이너에 `-v /var/run/docker.sock:/var/run/docker.sock`을 붙이면 그 컨테이너가 호스트의 Docker 전체를 부린다. net namespace로 네트워크를 갈라 놓아도 **파일은 마운트로 건너간다.** 관리 도구가 이걸 요구하면 정말 필요한지 따져 본다.
- 반대로 이 성질은 장점이기도 하다. 같은 서버 안의 DB는 TCP 포트를 열지 않고 소켓 파일로만 붙게 하면, 밖에서는 아예 닿을 수 없다.

방화벽 규칙도 비슷한 길로 커널에 들어갑니다. `ip`나 `nft` 명령은 커널과 **netlink 소켓**으로 대화합니다. 파일도 포트도 아닌, 사용자 프로그램과 커널 사이 전용 소켓입니다. 리눅스에서 "대화"는 거의 언제나 소켓이고, 그중 대부분은 포트 번호가 없습니다.

### 프로세스는 복제로 태어난다 — fork, exec, wait

셸에서 `sleep 3`을 치면 이런 일이 일어납니다.

```
bash (PID 100)
  │ ① fork  — 자기 자신을 복제한다 → bash (PID 101), 부모는 100
  │ ② exec  — 101번이 자기 몸을 /usr/bin/sleep 프로그램으로 갈아 끼운다
  │ ③ wait  — 100번(셸)은 101번이 끝나길 기다린다
  ▼
101번 종료 → 셸이 다시 프롬프트를 띄운다
```

POSIX의 `fork`는 지금 프로세스를 **메모리까지 통째로 복제**합니다(실제로는 바로 복사하지 않고, 둘 중 하나가 쓰려고 할 때 그 부분만 복사하는 copy-on-write라서 빠릅니다). 그다음 `exec`가 그 복제본의 내용을 **새 프로그램으로 갈아 끼웁니다.** 프로그램을 띄우는 일이 언제나 "복제 → 갈아 끼우기"라서, 모든 프로세스에는 **부모**가 있고, 그래서 프로세스는 **트리**가 됩니다.

```bash
# 지금 셸의 PID와 그 부모
echo "나: $$, 부모: $PPID"
# 셸 안에서 셸을 하나 더 — 새 PID, 부모는 아까 그 셸
bash -c 'echo "나: $$, 부모: $PPID"'
# 백그라운드로 하나 띄우고 트리로 보기
sleep 30 &
ps -o pid,ppid,cmd --forest
```

여기서 재미있는 결론 하나. **`cd`는 프로그램이 아닙니다.** 자식 프로세스는 자기 작업 디렉터리만 바꿀 수 있고 부모의 것은 못 바꿉니다. `cd`가 `/usr/bin/cd`라는 프로그램이었다면 fork된 자식이 디렉터리를 바꾸고 죽을 뿐, 셸은 그대로였을 겁니다. 그래서 `cd`·`export`는 셸 **안에** 들어 있는 내장 명령입니다.

```bash
# builtin — 셸 내장
type cd
# /usr/bin/ls — 디스크의 프로그램
type ls
```

**BusyBox — 명령 300개가 파일 하나.** 임베디드 장비와 경량 컨테이너(Alpine 등)에서는 `ls`·`cp`·`sh`·`ip` 같은 흔한 도구를 따로따로 두지 않고, **BusyBox라는 실행 파일 하나**에 모두 넣습니다. 각 명령은 그 파일을 가리키는 링크이고, BusyBox는 자기가 **어떤 이름으로 불렸는지**(`argv[0]`)를 보고 그 명령처럼 동작합니다.

```bash
docker run --rm alpine:3.20 sh -c 'ls -l /bin/ls /bin/sh /sbin/ip; busybox --list | wc -l; ls -l /bin/busybox'
```

```text
lrwxrwxrwx  1 root  root      12 … /bin/ls -> /bin/busybox
lrwxrwxrwx  1 root  root      12 … /bin/sh -> /bin/busybox
lrwxrwxrwx  1 root  root      12 … /sbin/ip -> /bin/busybox
304
-rwxr-xr-x  1 root  root  808712 … /bin/busybox
```

800 KB 파일 하나에 명령 304개 — Alpine 이미지 전체가 12 MB인 비결입니다. 그럼 fork는 안 일어날까요? 확인해 봅시다. `/proc/self`는 "지금 이걸 읽는 프로세스 자신"을 가리킵니다.

```bash
# 셸의 PID와, 셸이 띄운 명령들의 PID
docker run --rm alpine:3.20 sh -c 'echo "셸: $$"; readlink /proc/self; readlink /proc/self; readlink /proc/self'
```

```text
셸: 1
7
8
1
```

명령마다 **새 PID**(7, 8)입니다. BusyBox도 여전히 fork → exec를 합니다(같은 파일을 다시 exec할 뿐). 그런데 **마지막 줄은 1** — 셸 자신의 PID입니다. 더 할 일이 없는 마지막 명령은 셸이 fork하지 않고 **자기 몸을 그 명령으로 바로 갈아 끼웁니다(exec만).** 돌아올 필요가 없으니 복제할 이유도 없는 것이죠. bash도 `bash -c`에서 똑같이 합니다. 컨테이너의 PID 1이 셸 스크립트 끝에서 `exec 앱`으로 앱에게 자리를 넘겨주는 관례도 같은 원리입니다. 아끼는 건 **디스크와 메모리**입니다. 실행 파일이 하나뿐이라 디스크에 한 번만 있고, 메모리에도 한 번만 올라가 여러 프로세스가 같이 씁니다. (BusyBox를 직접 빌드할 때 일부 간단한 명령을 fork 없이 셸 안에서 돌리게 하는 옵션도 있지만, Alpine의 기본 설정에서는 위처럼 fork합니다.)

> ⚠️ 경량 이미지에서 "명령은 있는데 옵션이 다르다"는 일이 생기는 이유이기도 합니다. BusyBox의 `ls`·`grep`·`ip`는 GNU 도구를 줄인 판이라 일부 옵션이 없습니다. 예를 들어 GNU `ip`의 짧은 출력 옵션 `ip -br addr`을 Alpine에서 치면 결과 대신 BusyBox의 사용법(`BusyBox v1.36.1 … multi-call binary`)이 나옵니다. 앞에서 본 `ip addr`도 BusyBox의 `ip`였습니다.

### 환경변수 — 자식에게 물려주는 쪽지

`exec`로 몸을 갈아 끼우면 부모에게서 복제한 메모리는 사라집니다. 그런데 딱 하나, 갈아 끼울 때 **함께 넘겨주는 쪽지 묶음**이 있습니다. 그게 **환경변수**입니다. 셸 변수 중 `export`한 것만 이 쪽지에 적힙니다.

```bash
A=hello
# 같은 셸 안에서는 둘 다 보인다
echo "[$A]"
# 자식 프로그램(bash -c)에게는? — export 전
bash -c 'echo "[$A]"'
export A
# export 후
bash -c 'echo "[$A]"'
```

```text
[hello]
[]
[hello]
```

- `A=hello`는 **이 셸의 메모리에만** 있는 셸 변수. `export A`를 하면 자식에게 넘길 **환경변수**가 된다.
- 쪽지는 **부모 → 자식 한 방향**이다. 자식이 `export`해도 부모에게는 돌아오지 않는다. 그래서 `~/.bashrc`의 `export`는 "앞으로 이 셸이 띄울 모든 프로그램에게"라는 뜻이다.
- 한 번만 넘기려면 명령 앞에 붙인다: `A=hello bash -c 'echo "[$A]"'`
- 환경변수 목록은 `env`, 셸 변수까지 전부는 `set`.

> 💭 `( echo "[$A]" )`처럼 괄호 서브셸에서는 export 안 한 `A`도 보인다. 괄호는 fork만 하고 exec는 하지 않아서, 복제된 메모리에 `A`가 그대로 있기 때문이다. 위의 fork·exec 구분을 이해했는지 스스로 확인해 보기 좋은 예다.

### PATH — `ls`는 어디서 찾나

`ls`를 쳤을 때 셸은 지금 디렉터리를 뒤지지 않습니다. **환경변수 `PATH`에 적힌 디렉터리들을 왼쪽부터 차례로** 뒤져서 처음 찾은 `ls`를 실행합니다.

```bash
# 콜론(:)으로 구분된 목록 — 한 줄에 하나씩
echo "$PATH" | tr ':' '\n'
# ls가 어느 디렉터리에서 발견되나 (같은 이름이 여럿이면 전부)
type -a ls
# 내 디렉터리를 맨 앞에 — 같은 이름이면 앞쪽이 이긴다
export PATH="$HOME/bin:$PATH"
```

- **윈도우와 같은 개념이다.** 윈도우의 *시스템 속성 → 환경 변수 → Path*가 바로 이것이다(구분자만 `;`). 다만 윈도우 `cmd`는 **현재 폴더를 먼저** 뒤지고, 리눅스는 **현재 디렉터리를 뒤지지 않는다.**
- 그래서 리눅스에서 지금 디렉터리의 스크립트는 `./script.sh`로 경로를 붙여야 실행된다. 누군가 `/tmp`에 가짜 `ls`를 놓아 두어도, 내가 `cd /tmp` 한 뒤 `ls`를 쳤을 때 그게 실행되지 않게 하려는 **보안상의 선택**이다.
- 같은 이유로 `PATH` 맨 앞에 남이 쓸 수 있는 디렉터리를 넣으면 안 된다.
- bash는 한 번 찾은 위치를 기억해 둔다(`hash`로 보인다). 프로그램을 옮겼는데 옛 위치를 찾으면 `hash -r`.

### 프로세스 트리를 보면 — 두 그루의 나무와 [대괄호]

```bash
ps auxf | head -20
```

맨 위에 이런 줄들이 보입니다.

```text
root   2  ...  [kthreadd]
root   3  ...   \_ [pool_workqueue_release]
root   4  ...   \_ [kworker/R-rcu_gp]
```

`ps auxf`에는 나무가 **두 그루** 있습니다.

- **PID 1 (`systemd`)** — 사용자 공간의 뿌리. 로그인 셸, 서비스, 컨테이너 런타임이 모두 이 아래로 이어진다.
- **PID 2 (`kthreadd`)** — **커널 스레드**의 뿌리. `[대괄호]`로 보이는 것들이 그 자식이다.

대괄호는 "**명령줄이 없다**"는 표시입니다. 커널 스레드는 디스크의 프로그램 파일을 `exec`해서 뜬 게 아니라 커널 안에서 커널이 직접 만든 작업자라서 `/proc/⟨PID⟩/cmdline`이 비어 있고, `ps`는 그럴 때 이름을 대괄호로 감쌉니다. 이름을 보면 무슨 일을 하는지 대략 보입니다.

| 커널 스레드 | 하는 일 |
| --- | --- |
| `[kworker/…]` | 커널의 잡일 대기열(workqueue) 처리 |
| `[ksoftirqd/0]` | 인터럽트 뒤처리 — 네트워크 패킷이 몰리면 이게 CPU를 먹는다 |
| `[kswapd0]` | 메모리가 모자랄 때 회수 |
| `[jbd2/vda1-8]` | ext4 파일시스템의 저널 기록 (`vda1`의 것) |

`PID 1`과 `PID 2`의 부모는 둘 다 **0**입니다. 0번은 부팅할 때의 커널 자신이라 목록에 나오지 않습니다. 예전 커널에서는 커널 스레드도 init(PID 1) 아래에 붙어 한 그루로 보였지만, 지금은 `kthreadd`가 모든 커널 스레드의 부모를 맡아서 두 그루로 나뉩니다.

systemd 이후에는 트리를 보는 눈이 하나 더 생겼습니다. 옛 init.d 스크립트의 데몬은 스스로 두 번 fork해서 부모를 떠나는 방식이라, 누가 무엇을 띄웠는지 트리만으로는 추적이 어려웠습니다. systemd는 서비스마다 **cgroup을 하나씩** 주고 그 안의 프로세스를 모두 추적합니다. 그래서 "이 프로세스가 어느 서비스 소속인가"는 `ps auxf`보다 `systemd-cgls`가 정확합니다. 컨테이너도 같습니다. 호스트에서 `ps auxf`를 보면 컨테이너의 프로세스가 `containerd-shim-runc-v2` 아래에 매달려 있고, 컨테이너 안에서 보면 그 프로세스가 **PID 1**입니다. 같은 프로세스가 pid namespace에 따라 다른 번호로 보이는 것입니다.

### 컨테이너의 환경변수 — `docker inspect`의 Env

컨테이너의 첫 프로세스(PID 1)도 결국 `exec`로 뜹니다. 그때 넘겨주는 쪽지 묶음이 `docker inspect`의 `Config.Env`입니다. 곧 **export된 환경변수 목록**입니다.

```bash
docker run -d --name pgenv -e POSTGRES_HOST_AUTH_METHOD=trust postgres:16
docker inspect pgenv --format '{{json .Config.Env}}'
docker rm -f pgenv
```

```text
["POSTGRES_HOST_AUTH_METHOD=trust","PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/16/bin","GOSU_VERSION=1.19","LANG=en_US.utf8","PG_MAJOR=16","PG_VERSION=16.15-1.pgdg13+2","PGDATA=/var/lib/postgresql/data"]
```

내가 `-e`로 준 것 하나에, **이미지의 Dockerfile `ENV`** 로 정해진 것들(`PATH`·`PG_VERSION` 등)이 더해져 있습니다. `PATH`에 `/usr/lib/postgresql/16/bin`이 들어 있어서 컨테이너 안에서 `psql`을 경로 없이 칠 수 있는 겁니다.

compose에서 컨테이너로 들어가는 길은 `environment:`와 `env_file:`입니다. ⚠️ **프로젝트 디렉터리의 `.env` 파일은 자동으로 컨테이너에 들어가지 않습니다.** `.env`는 `compose.yaml` 안의 `${TAG}` 같은 자리를 채우는 데 쓰입니다. 직접 해 보면 이렇습니다.

```bash
mkdir envtest && cd envtest
printf 'FOO=from_dotenv\nTAG=3.20\n' > .env
cat > compose.yaml <<'EOF'
services:
  app:
    image: alpine:${TAG}
    command: sleep 300
    environment:
      BAR: "set_in_compose"
EOF
docker compose up -d
# 예측: FOO와 BAR가 둘 다 보일까?
docker inspect envtest-app-1 --format '{{json .Config.Env}}'
docker compose down -t 0
```

```text
["BAR=set_in_compose","PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"]
```

`TAG`는 이미지 태그(`alpine:3.20`)를 채우는 데 쓰였지만 `FOO`는 컨테이너에 없습니다. `.env`의 값을 컨테이너에 넣으려면 `env_file: .env`를 쓰거나 `environment:`에 `FOO: ${FOO}`처럼 명시합니다.

⚠️ 환경변수는 **비밀을 두기에 좋은 곳이 아닙니다.** `docker` 권한이 있는 사람은 누구나 `docker inspect`로 평문을 보고, 컨테이너 안에서는 `/proc/1/environ`으로 보입니다. 비밀번호·토큰은 읽기 전용으로 마운트한 파일로 넘기는 편이 낫습니다(많은 공식 이미지가 `POSTGRES_PASSWORD_FILE`처럼 `_FILE` 변수를 지원합니다).

더 깊이: [Kadav & Swift, 「Understanding Modern Device Drivers」, ASPLOS 2012](https://pages.cs.wisc.edu/~kadav/app/papers/driverstudy.pdf), [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html), [cgroups(7)](https://man7.org/linux/man-pages/man7/cgroups.7.html), [fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html), [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html), [environ(7)](https://man7.org/linux/man-pages/man7/environ.7.html), [Docker Compose: 환경변수 우선순위와 .env](https://docs.docker.com/compose/how-tos/environment-variables/).

## 4.1 컨테이너 vs VM — 과잉설계 경계

**멘탈모델.** VM은 *컴퓨터 한 대*를 흉내 낸다(커널까지 따로). 컨테이너는 *호스트 커널 위의 격리된 프로세스*다(namespace + cgroup). 그래서 VM은 "OS를 운영"하고, 컨테이너는 "프로그램 하나를 운영"한다.

| | VM | 컨테이너 |
|---|---|---|
| 격리 단위 | 커널 포함 한 대 | 프로세스(호스트 커널 공유) |
| 기동 | 분 단위 | 초 단위 |
| 격리 강도 | 강함(하이퍼바이저 경계) | 상대적으로 약함(커널 공유) |
| 산출물 | 디스크 이미지(수 GB) | 레이어 이미지(수십~수백 MB) |
| 잘 맞는 일 | 다른 OS, 강한 멀티테넌트 | 서비스 하나, 재현 가능한 배포 |

**왜 이게 중요한가.** 커널을 공유한다는 건 *컨테이너 탈출 = 호스트 장악*이라는 뜻이다. 그래서 신뢰할 수 없는 코드를 돌릴 땐 VM 경계가 여전히 낫고, 컨테이너에는 비루트(non-root: root가 아닌 일반 사용자로 실행)·egress 차단 같은 보강이 붙는다(4.4, 4.5).

**과잉설계 경계.** 컨테이너가 쉬워지면 반대 실수가 온다. "서비스 헬스 보고 싶다"에 Prometheus + Grafana + Alertmanager + exporter를 짓는 것.

- 필요가 *"죽었는지 알림"* 이면 → 단일 컨테이너(예: Uptime Kuma)로 끝.
- 필요가 *"수백 개 지표의 시계열 분석"* 일 때 비로소 풀스택.
- 판단 기준: **내가 6개월 뒤에도 혼자 고칠 수 있는가.** 컨테이너 수 = 유지보수 부담 수.

이 판단은 머리에만 두지 말고 ADR 한 장으로 남긴다([실습⑨](labs/lab-09.md)의 "대안" 칸). 과잉설계는 대개 *대안을 적지 않아서* 생긴다.

**확인.**

```bash
# 내가 몇 개를 돌리고 있나
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
# 그 값어치를 하나
docker stats --no-stream
# 같은 커널 = 공유의 증거
uname -r; docker exec <ctr> uname -r
```

**바깥 링크.** [Docker: What is a container](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/) · [Linux namespaces(man7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)

## 4.2 선언형 — compose, "뭘 깔았더라"가 사라진다

**멘탈모델.** 명령형은 *"무엇을 했는가"* 의 기록(셸 히스토리, 기억), 선언형은 *"무엇이어야 하는가"* 의 기록(파일)이다. 서버 상태 = `compose.yaml` + 데이터 볼륨. 그 외는 전부 버려도 되는 것이다.

```yaml
# compose.yaml — 이 한 파일이 "설치 문서"를 대신한다
services:
  kuma:
    # 버전을 고정(재현성)
    image: louislam/uptime-kuma:1
    restart: unless-stopped
    # 바깥에 열지 않고 로컬만
    ports: ["127.0.0.1:3001:3001"]
    volumes:
      # 지켜야 할 건 이것뿐
      - kuma-data:/app/data
volumes:
  kuma-data: {}
```

**왜.**

- **재현성:** 서버가 망가져도 지우고 `up`하면 복구된다. "복구 절차"가 곧 "설치 절차"라 따로 문서화할 게 없다.
- **감사 가능:** 파일이 git에 있으면 *누가 언제 왜* 바꿨는지가 남는다. 섀도우 IT가 양지로 나오는 지점이다.
- **데이터와 분리:** 컨테이너는 소모품, 볼륨만 자산이다. 백업 대상이 명확해진다([Part 1](01-image.md)·백업 장 참고).

**확인.** 복구 리허설을 한 번은 해본다.

```bash
docker compose up -d && docker compose ps
# 컨테이너·네트워크 삭제 (볼륨은 유지)
docker compose down
# 그대로 복구 — 데이터도 살아 있다
docker compose up -d
# -v 는 볼륨까지 삭제: 되돌릴 수 없음
docker compose down -v
# 변수 치환이 끝난 최종 선언을 보기
docker compose config
```

> 함정: `latest` 태그와 이름 없는 볼륨은 선언형을 조용히 망친다. 버전은 고정하고, 지킬 데이터는 이름 있는 볼륨이나 명시적 경로로 둔다.

**바깥 링크.** [Compose specification](https://docs.docker.com/reference/compose-file/) · [Docker: Volumes](https://docs.docker.com/engine/storage/volumes/)

## 4.3 손에 익히기 — exec·alias·설정 추출/마운트·패키징

선언형이어도 손은 쓴다. 네 가지 손놀림이 컨테이너를 "남의 서비스"에서 "내 도구"로 바꾼다. 따라 하기는 [실습⑨ 보강](labs/lab-09.md)에 있고, 여기선 *왜 그렇게 하는지*만 짚는다.

### (1) `docker exec -it` — 컨테이너 안으로

```bash
# compose 서비스 이름으로
docker compose exec app sh
# 컨테이너 이름으로 (쉘이 없으면 sh)
docker exec -it <ctr> bash
```

- `exec`는 *이미 도는* 컨테이너의 네임스페이스에 새 프로세스를 끼워 넣는다. `run`은 *새* 컨테이너를 만든다. 혼동하면 "고쳤는데 사라졌다"가 된다.
- 안에서 고친 파일은 컨테이너와 함께 사라진다. **구경과 디버깅에만** 쓰고 수정은 4.3 (3)으로.
- 보안: `docker` 그룹 = 사실상 호스트 root. `exec` 권한은 곧 그 컨테이너의 권한이다([Part 5](05-trust.md)·AI 에이전트에 줄 때 특히 유의).

### (2) `alias` — 컨테이너를 로컬 도구함으로

```bash
alias psql='docker compose exec -T db psql -U postgres'
alias redis-cli='docker exec -it redis redis-cli'
# 로컬에 설치한 것처럼
psql -c 'select now();'
```

- 호스트에 DB 클라이언트를 *설치하지 않는다.* 버전도 서버와 같은 이미지에서 온다 — 클라이언트/서버 버전 불일치가 사라진다.
- 파이프·리다이렉트를 쓸 땐 `-T`(또는 `-i`만), 대화형이면 `-it`. TTY 유무가 가장 흔한 실패 원인이다.
- `~/.zshrc`에 넣어 두면 장비마다 따라온다.

### (3) 설정: 추출 → 수정 → 볼륨 마운트 ("굽지 말고 마운트")

```bash
# 1) 기본값 추출
docker run --rm nginx cat /etc/nginx/nginx.conf > ./nginx.conf
# 2) 수정
vi ./nginx.conf
```

```yaml
services:
  web:
    image: nginx:1.27
    volumes:
      # 3) 덮어 마운트
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

- 이미지는 **불변**, 바뀌는 것은 **밖에서 주입**. 이미지를 업그레이드해도 내 설정은 그대로다.
- 설정을 이미지에 `COPY`로 굽는 건 *설정이 바뀔 때마다 빌드*가 필요해 느리고, 비밀(키·비밀번호)이 레이어에 남을 위험도 있다.
- `:ro`로 마운트하면 컨테이너가 털려도 설정을 못 바꾼다.

### (4) Dockerfile — 내 앱 패키징

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 1000:1000
CMD ["python", "app.py"]
```

- "내 PC에서는 되는데"가 이미지 한 덩어리로 해결된다. 다른 서버에선 `docker run` 한 줄이다.
- 의존성 파일을 먼저 `COPY`하는 순서는 *캐시* 때문이다(4.5).

**확인.**

```bash
# 지금 누구로 도나 (root면 4.5로)
docker compose exec app id
docker build -t myapp:1.0 . && docker run --rm -p 8000:8000 myapp:1.0
```

**바깥 링크.** [docker exec](https://docs.docker.com/reference/cli/docker/container/exec/) · [Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)

## 4.4 네트워크 멤버십 = egress 허용 목록

**멘탈모델.** 컨테이너의 "나갈 수 있는 곳"은 방화벽 규칙이 아니라 **어느 네트워크에 붙어 있느냐**로 정해진다. compose의 `networks:` 목록이 곧 *허용 목록*이다. 이것이 SADP의 outbound 통제를 백엔드에 적용한 모습이다([실습⑨′](labs/lab-09p.md)).

```text
              인터넷
                 |
   [edge]  ──── caddy ──── [web, internal] ──── app ──── [data, internal] ──── db
 (egress 가능)                (인터넷 차단)                  (인터넷 차단)
```

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["443:443"]
    # 바깥과 app 사이의 문지기
    networks: [edge, web]
  app:
    build: .
    # 붙은 곳 = 말할 수 있는 곳 (edge 없음)
    networks: [web, data]
  db:
    image: postgres:16
    networks: [data]
networks:
  # 기본 bridge: NAT로 외부 나감
  edge: {}
  web:  { internal: true }
  data: { internal: true }
```

**왜.** 백엔드가 털렸을 때의 피해는 대부분 *나가는 길*에서 결정된다. C2 통신, 데이터 유출, 추가 도구 다운로드가 전부 outbound다. 앱이 인터넷에 갈 일이 없다면 길 자체를 없앤다. 방화벽 규칙을 쓰지 않아도 되고, 선언이 곧 정책이라 *리뷰 가능*하다.

`internal: true`는 Docker가 그 네트워크에 외부 라우팅·NAT(masquerade)를 만들지 않는다는 뜻이다. 게이트웨이가 없으니 `8.8.8.8`로 갈 길이 없다.

**확인.** 가두기 전/후를 반드시 대조한다.

```bash
# internal 적용 후: 100% loss
docker compose exec app ping -c3 8.8.8.8
# OK (web 공유)
docker compose exec app ping -c2 caddy
# OK (data 공유)
docker compose exec app ping -c2 db
# OK (edge)
docker compose exec caddy ping -c2 8.8.8.8
# true
docker network inspect <project>_web --format '{{.Internal}}'
```

> `ping`이 없으면 `getent hosts caddy`, `curl -m3 https://example.com`로 대체한다.

**한계 — 이것이 전부는 아니다.**

- `internal`은 "인터넷 차단"이지 "그 네트워크 *안*까지 차단"이 아니다. 같은 net의 컨테이너끼리는 서로 모든 포트로 통한다. 예: app이 털리면 같은 `data` net의 db 전 포트가 열려 있다.
- 서비스를 `edge`에 붙이는 순간 그 컨테이너는 *전면 egress 허용*이다. 붙이는 곳을 최소화하고, caddy 같은 문지기만 붙인다.
- 목적지·포트 단위로 조이려면 호스트의 `DOCKER-USER` 체인으로 내려간다 → [Part 2](02-firewall.md). 컨테이너 트래픽이 `INPUT`이 아니라 `FORWARD`를 지나므로 일반 방화벽 규칙이 안 먹는 이유도 거기서 다룬다.

**같은 생각, 여러 고도.** 호스트(실습⑥) → **컨테이너(여기)** → 망 경계(실습⑧) → AI 에이전트(실습⑭). 전부 "나가는 길은 기본 차단, 필요한 곳만 허용"이다.

**바깥 링크.** [Docker: Networking](https://docs.docker.com/engine/network/) · [Compose: networks (`internal`)](https://docs.docker.com/reference/compose-file/networks/#internal) · [Packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

## 4.5 이미지 빌드 — Dockerfile, 비루트(non-root), 멀티스테이지, 대역 겹침

### 비루트(non-root) — `USER`

**멘탈모델.** 컨테이너 안의 root는 호스트 root와 *같은 UID 0*이다. 격리가 뚫리거나 볼륨을 마운트한 상태라면 그 힘이 그대로 호스트에 닿는다. 앱은 root일 이유가 없다.

```dockerfile
RUN useradd --uid 10001 --no-create-home app
USER 10001:10001
```

- 숫자 UID로 지정하면 오케스트레이터가 "비루트인지" 검증하기 쉽다.
- 1024 미만 포트는 못 연다 → 8080처럼 높은 포트에서 받고 앞단(caddy)이 443을 맡는다.
- 마운트한 볼륨의 소유권이 UID와 맞아야 쓰기가 된다. "Permission denied"의 대부분이 이것이다.

```bash
# uid=10001 이어야 함
docker compose exec app id
# 비면 root
docker inspect -f '{{.Config.User}}' myapp:1.0
```

### 멀티스테이지

**멘탈모델.** *빌드 도구는 빌드에만, 실행에는 결과물만.* 컴파일러·패키지 매니저·소스는 공격 표면이자 용량이다. 최종 이미지에 넣지 않는다.

```dockerfile
# --- 빌드 단계: 도구 잔뜩 ---
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
# 의존성 먼저 → 캐시 적중
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

# --- 실행 단계: 결과물만 ---
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

- 결과 이미지에는 쉘도 `ping`도 없다. 디버깅이 불편한 대신 털려도 쓸 도구가 없다. 트레이드오프를 알고 고른다(실습⑨′은 `ping` 확인용으로 일부러 넣은 것).
- 레이어는 순서가 캐시다. *자주 바뀌는 것을 아래에* 둔다. 일반론은 아래 표준 문서로.

### 컨테이너 대역이 기관망·VPN과 겹치지 않게 — `default-address-pools`

**멘탈모델.** Docker는 네트워크를 만들 때마다 기본 풀(`172.17.0.0/16`부터 `172.31.x`, 이어서 `192.168.x`)에서 대역을 *자동* 할당한다. 기관 내부망이나 VPN이 같은 대역을 쓰면, 호스트는 그쪽 목적지를 *컨테이너 브리지로 보내버린다.* 증상은 "컨테이너를 띄우자 VPN/사내 서버 접속이 끊긴다", 또는 반대로 컨테이너가 사내 서버에 못 간다.

원인은 **라우팅 테이블에서 더 구체적인(또는 나중에 생긴) 경로가 이긴다**는 것뿐이다. 컨테이너 대역과 겹치는 목적지는 방화벽 이전에 라우팅 단계에서 이미 갈 곳을 잃는다.

**확인.** 먼저 지금 무엇이 겹치는지 본다.

```bash
# 사내망/VPN 경로와 docker 브리지 경로 비교
ip route
docker network ls -q | xargs docker network inspect \
  --format '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

**처방.** 기관이 쓰지 않는 대역을 *미리 한 번* 정해 두고, 풀을 그 안으로 제한한다.

```json
// /etc/docker/daemon.json
{
  "bip": "172.31.255.1/24",
  "default-address-pools": [
    { "base": "172.31.0.0/16", "size": 24 }
  ]
}
```

- `bip`: 기본 브리지 `docker0`의 대역. `default-address-pools`: compose 등이 만드는 *사용자 정의* 네트워크의 풀(`base`를 `size` 크기로 잘라 쓴다).
- 위 예(`172.31.0.0/16`, `/24`씩)는 네트워크 약 255개분이다. 프로젝트가 많으면 `size`를 줄이거나 base를 넓힌다.
- **적용:** `sudo systemctl restart docker`. 이미 만든 네트워크는 그대로이므로 `docker compose down && up -d`로 재생성한다. 재시작은 컨테이너를 멈추니 점검 창에서 한다.
- 대역은 **기관 IP 담당과 합의한 값**을 쓴다. 이 예시는 문서용이다.
- firewalld를 쓰는 서버라면 브리지 인터페이스가 어느 zone에 속하는지도 함께 확인한다. Docker 트래픽이 firewalld와 *다른 길*로 흐르는 이유는 [Part 2](02-firewall.md)에서 다룬다.

**바깥 링크.** [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/) · [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/) · [Daemon configuration (`default-address-pools`)](https://docs.docker.com/engine/daemon/) · [Docker and firewalld](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-firewalld)
