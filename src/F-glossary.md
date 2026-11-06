# 부록 F. 용어집 {#appendix-f}

숫자·기호, 한글(가나다순), 영문(ABC순) 차례입니다. 각 용어가 본문 어디에 나오는지는 [찾아보기](Z-index.md)에서 볼 수 있습니다.

| 용어 | 설명 |
|---|---|
| 2FA | 2단계 인증. 비밀번호에 휴대전화·보안키 같은 두 번째 수단을 더한다. |
| 802.1X | 포트 단위 인증 표준. 인증된 단말만 네트워크에 붙게 한다(NAC의 구현 수단). |
| 골든 이미지 (golden image) | 표준으로 삼아 복제해 쓰는 기준 디스크 이미지. → [Part 1 §1.1](01-image.md#1-1-golden-image) |
| 공격면 (attack surface) | 외부에서 공격받을 수 있는 지점의 총합. 열린 포트·노출된 서비스가 넓힐수록 커진다. |
| 굽다 (bake) | 설정이나 파일을 넣어 이미지를 새로 빌드하는 것. 반대는 실행할 때 마운트하는 것이다. → [Part 4 §4.3](04-container.md#4-3-config-mount) |
| 드롭인 (drop-in) | 본 설정 파일을 고치지 않고 `*.d/` 디렉터리에 조각 파일을 넣어 설정을 덧붙이는 방식. 패키지 업데이트에도 설정이 유지된다. → [Part 1 §1.3](01-image.md#1-3-drop-in) |
| 루트 프로그램 | OS·브라우저 회사가 어떤 루트 CA를 기본으로 믿을지 정하고 관리하는 제도(Mozilla·Microsoft·Apple·Chrome). 목록은 설치 미디어와 OS 업데이트로 배포된다. → [Part 5 §5.1](05-trust.md) |
| 리버스 프록시 (reverse proxy) | 외부 요청을 대신 받아 뒤쪽 서버에 전달하는 서버. HTTPS·로그인을 앞단에서 처리할 때 쓴다(예: Caddy). |
| 링크드 클론 (linked clone) | 원본 디스크를 읽기 전용 백킹 파일로 두고 바뀐 블록만 따로 저장하는 복제. 빠르고 작지만 원본을 지우면 함께 망가진다. → [Part 1 §1.2](01-image.md#1-2-image-formats) |
| 멀티스테이지 빌드 | Dockerfile 안에 여러 단계(`FROM`)를 두고, 앞 단계의 결과물만 최종 이미지로 옮기는 방식. → [Part 4 §4.5](04-container.md#4-5-multi-stage) |
| 멀티홈드 (multi-homed) | 두 개 이상의 망에 동시에 연결된 호스트. default route는 하나만 둔다. → [Part 3 §3.2](03-network.md#3-2-multi-homed) |
| 멘탈모델 (mental model) | 대상이 어떻게 동작하는지 단순하게 잡아 둔 이해의 틀. 이 책은 각 절 첫머리에 한두 문장으로 적는다. → [들어가며](00-intro.md#how-to-use) |
| 멱등 (idempotent) | 몇 번을 실행해도 결과가 같은 성질. 설정 스크립트가 갖춰야 할 성질이다. |
| 미러 (mirror) | 원본 저장소를 복제해 제공하는 서버. |
| 방탄 호스팅 (bulletproof hosting) | 신고·차단 요청에 응하지 않는 호스팅. 평판 목록이 대역째로 막는 대상이다. |
| 섀도우 IT | 조직의 승인·관리 밖에서 현업이 스스로 마련해 운영하는 시스템과 서비스. → [Part 7 §7.5](07-ai-governance.md#7-5-governance) |
| 선언형 운영 | 원하는 최종 상태를 파일로 적어 두고 도구가 그 상태를 맞추게 하는 운영 방식. → [Part 4 §4.2](04-container.md#4-2-compose) |
| 스테이징 (staging) | 운영에 반영하기 전에 같은 구성으로 시험하는 환경. |
| 스푸핑 (spoofing) | 출발지 주소 등을 위조하는 공격. |
| 암묵지 | 문서로 정리되지 않고 사람에게만 남아 있는 노하우. → [Part 1 §1.3](01-image.md#1-3-dotfiles) |
| 오케스트레이터 | 쿠버네티스처럼 여러 컨테이너를 여러 서버에 배치·관리하는 시스템. |
| 오탐 | 정상 대상을 악성으로 잘못 판정하는 것(false positive). |
| 위임 사다리 | AI에게 맡기는 정도를 조언→조사→승인 후 실행→위임의 네 칸으로 나눈 것. 칸마다 계정·권한·사람의 확인 방법이 다르다. → [Part 7 §7.1](07-ai-governance.md) |
| 인벤토리 | 서버·소프트웨어·계정 같은 자산과 그 식별 정보를 모은 목록. → [Part 7 §7.2](07-ai-governance.md#7-2-inventory) |
| 제로 트러스트 | "내부망이니 믿는다"를 버리고 연결마다 주체를 확인해 필요한 만큼만 허락하는 보안 방식(NIST SP 800-207). mTLS가 대표적 구현 수단. → [Part 5 §5.6](05-trust.md) |
| 초과 약정 (over-provisioning) | 실제 저장 용량보다 많은 용량을 VM들에 약속해 두는 것. thin 디스크에서 생기며, 실제로 다 쓰면 저장소가 가득 찬다. → [Part 1 §1.2](01-image.md#1-2-image-formats) |
| 커널 스레드 | 커널 안에서 커널이 직접 만든 작업자. `ps`에서 `[kworker/…]`처럼 대괄호로 보이고 부모는 `kthreadd`(PID 2). → [Part 4 §4.0](04-container.md) |
| 키스토어 (keystore) · 키체인 | 기기가 신뢰하는 인증서와 개인키를 보관하는 OS의 저장소. → [Part 5 §5.4](05-trust.md#5-4-keystore) |
| 토폴로지 | 망의 연결 구조. 어떤 장비가 어느 망에 붙어 있는지의 배치. |
| 투명 브리지 (transparent bridge) | IP 주소 없이 두 망 사이에 끼워 패킷을 걸러 내는 장치. → [Part 2 §2.5](02-firewall.md#2-5-case) |
| 패스프레이즈 (passphrase) | 개인키 파일을 암호화하는 긴 암호. |
| 패키지 관리자 | 정해진 저장소에서 프로그램을 받아 서명 검증·의존성 설치·업데이트·제거를 해 주는 도구(apt, dnf, Homebrew, WinGet). 설치 목록을 파일로 내보내 다시 설치할 수 있다. → [부록 G](G-packages.md) |
| 폭발 반경 (blast radius) | 사고 하나가 영향을 미치는 범위. 격리의 목적은 이것을 줄이는 것이다. |
| 프로비저닝 (provisioning) | 서버를 쓸 수 있는 상태로 설치·설정을 미리 마쳐 두는 작업. |
| 프록시 (proxy) | 클라이언트 대신 외부에 접속해 요청을 중계하는 서버. 나가는 연결을 한곳으로 모을 때 쓴다. → [Part 2 §2.6](02-firewall.md#2-6-egress-allowlist) |
| 호스트명 규칙 | VM 구분용 이름 짓기 예: `⟨IP 끝자리 3자리⟩-⟨OS약어⟩-⟨용도⟩` (예: `042-u24-web`). |
| 화이트리스팅 (allowlist) | 허용 목록에 있는 대상만 통과시키고 나머지는 막는 방식. → [Part 2 §2.6](02-firewall.md#2-6-egress-allowlist) |
| 환경변수 (`export`) | 프로세스가 자식에게 물려주는 이름=값 목록. 셸 변수는 `export`해야 자식 프로그램에 넘어간다. → [Part 4 §4.0](04-container.md) |
| 훅 (hook) | 커널 안에서 패킷이 방화벽 규칙의 검사를 받는 지점(prerouting, input, forward 등). → [Part 2 §2.2](02-firewall.md#2-2-hooks) |
| ACME | 인증서를 자동 발급·갱신하는 프로토콜(RFC 8555). Let's Encrypt가 사용. |
| ADR | Architecture Decision Record. 결정과 이유를 짧게 남기는 문서. → [부록 C](C-adr.md) |
| authorized_keys | 서버의 `~/.ssh/authorized_keys`. 이 계정에 들어올 수 있는 **공개키**를 한 줄에 하나씩 적는 파일(디렉터리 700·파일 600). → [Part 6 §6.1](06-access.md) |
| bastion | 내부망 진입 전용 관문 서버. 외부 접속을 여기 한 곳으로 모은다. |
| BMC (Baseboard Management Controller) | 서버 본체와 별도로 동작하며 원격 관리(IPMI)를 맡는 컨트롤러. |
| C2 (Command and Control) | 공격자가 침해한 서버를 원격으로 조종하는 명령·제어 서버. 나가는 연결을 막으면 연결할 수 없다. |
| CA | 인증기관. 서버·클라이언트 인증서에 서명해 신뢰를 보증한다. |
| Caddy | HTTPS가 기본값인 오픈소스 웹 서버·리버스 프록시. 인증서 발급·갱신·리다이렉트를 자동으로 한다. → [Part 5 §5.2](05-trust.md) |
| CC 인증 (Common Criteria) | 정보보호 제품을 평가하는 국제 공통평가기준에 따른 인증. |
| CCE | Common Configuration Enumeration. 보안 설정 항목의 표준 식별자. |
| CDN (Content Delivery Network) | 사용자와 가까운 서버에서 콘텐츠를 대신 제공하는 서비스. |
| Ceph | 여러 서버의 디스크를 묶어 쓰는 분산 스토리지 시스템. OSD는 실제로 데이터를 저장하는 Ceph 데몬이다. |
| cgroup | 프로세스 묶음별로 CPU·메모리·I/O·프로세스 수를 제한하고 재는 리눅스 기능. 컨테이너의 "쿼터" 쪽. → [Part 4 §4.0](04-container.md) |
| CN (Common Name) | 인증서 주체(Subject)의 이름 필드. mTLS에서 권한을 나누는 조건으로 쓸 수 있다. → [Part 5 §5.6](05-trust.md#5-6-zero-trust) |
| CSR (Certificate Signing Request) | 인증서 서명 요청. 공개키와 주체 정보를 담아 CA에 보낸다. |
| CVE | 공개된 취약점의 표준 식별 번호(예: CVE-2024-xxxx). |
| deb822 | `키: 값` 줄로 항목을 적는 APT 저장소 파일 형식(`.sources`). PVE 9·Debian 13부터 기본이다. → [부록 H.2.3](H-pve.md#h-2-3-packages) |
| digest (이미지) | 컨테이너 이미지 내용으로 계산한 해시값(`sha256:…`). 태그와 달리 바뀌지 않으므로 고정에 쓴다. |
| DNAT | 목적지 주소 변환. 외부 포트를 내부 서비스로 포워딩할 때 쓴다. |
| DNS 존 (zone) | 한 도메인 아래의 DNS 레코드를 묶어 관리하는 단위. → [Part 5 §5.2](05-trust.md#5-2-least-privilege) |
| DNS-01 | ACME 검증 방식. DNS TXT 레코드로 소유를 증명한다(와일드카드 가능). |
| DOCKER-USER | Docker가 FORWARD 앞에 두는 사용자 정의 iptables 체인. 컨테이너 방화벽을 여기서 건다. |
| dotfile (점 파일) | 이름이 `.`으로 시작하는 숨김 파일(`ls -a`로 보임). `~/.bashrc`·`~/.ssh/config` 같은 사용자 설정. 팀이 공유하면 온보딩과 편차 줄이기에 효과적이다. → [Part 1 §1.3](01-image.md) |
| DPI (Deep Packet Inspection) | 패킷의 내용까지 들여다보는 검사. HTTPS에 쓰려면 기관 CA를 기기에 심고 중간에서 복호화한다. → [Part 5 §5.5](05-trust.md#5-5-dpi) |
| egress | 나가는 트래픽. "어느 고도에서 egress를 통제하나"가 이 책의 한 줄. |
| fork / exec | 프로세스가 자신을 복제(fork)한 뒤 새 프로그램으로 갈아 끼우는(exec) 유닉스의 프로그램 실행 방식. 그래서 프로세스에는 부모가 있고 트리가 된다. → [Part 4 §4.0](04-container.md) |
| forward-auth | 리버스 프록시가 요청을 인증 서버에 먼저 물어보고 통과시키는 방식. |
| FUSE (Filesystem in Userspace) | 커널이 아닌 사용자 공간 프로그램으로 동작하는 파일시스템. PVE의 `/etc/pve`가 이 방식이다. |
| HNUM | ohmyzsh 프롬프트에 표시되는 셸 히스토리 번호. `!N`으로 그 번호의 명령을 다시 실행한다. |
| HTTP-01 | ACME 검증 방식. 80포트의 지정 경로로 소유를 증명한다. |
| IdP (Identity Provider) | 사용자 인증(로그인)을 맡는 계정 서버. OIDC 로그인의 상대편이다. → [Part 6 §6.4](06-access.md#6-4-idp) |
| INPUT / FORWARD | iptables/nftables 체인. INPUT은 호스트 자신에게 오는 것, FORWARD는 호스트를 거쳐 가는 것(컨테이너 트래픽). |
| internal network | 외부로 나가는 경로가 없는 Docker 네트워크(`internal: true`). |
| IPMI | 서버 보드(BMC)의 원격 관리 인터페이스. 전원·콘솔 제어. 외부에 노출하지 말 것. |
| IPSet (Proxmox) | 이름을 붙여 방화벽 규칙에서 참조하는 주소 목록. `management`는 관리 접속을 허용하는 특별한 IPSet이다. → [부록 H.4](H-pve.md#h-4-ipset) |
| Let's Encrypt | 비영리 단체 ISRG가 운영하는 무료 공인 CA(2015~). ACME로 자동 발급·갱신하며, 글로벌 기업과 EFF·Mozilla 등의 후원으로 운영된다. → [Part 5 §5.2](05-trust.md) |
| LSM (Linux Security Module) | SELinux·AppArmor처럼 커널에 보안 검사를 끼워 넣는 틀. |
| LUKS | 리눅스 디스크 암호화 표준. 분실·도난 시 데이터 보호. |
| LXC | 호스트 커널을 공유하는 리눅스 컨테이너. PVE가 VM과 함께 지원한다. |
| machine-id | `/etc/machine-id`. 머신 고유 ID로, 이미지 복제 시 반드시 리셋해야 한다. |
| MADR | Markdown Architecture Decision Records. 마크다운으로 쓰는 ADR 양식. → [부록 C](C-adr.md) |
| mTLS | 서버뿐 아니라 클라이언트도 인증서로 인증하는 상호 TLS. |
| NAC | Network Access Control. 단말 인증·상태 검사 후 네트워크 접근 허용. |
| namespace | 프로세스가 보는 세상(PID·네트워크·마운트·호스트명·사용자)을 따로 떼어 주는 리눅스 기능. 컨테이너의 "격리" 쪽. → [Part 4 §4.0](04-container.md) |
| nftables set | 여러 주소·대역을 하나로 묶어 한 번에 조회하는 nftables의 자료 구조. 규칙 수천 개 대신 쓴다. → [Part 2 §2.5](02-firewall.md#2-5-why-set) |
| non-root (비루트) | 프로세스를 root가 아닌 일반 사용자로 실행하는 것. 뚫려도 피해가 그 사용자 권한에 머문다. Dockerfile의 `USER`. → [Part 4 §4.5](04-container.md) |
| ohmyzsh 태그 (TAG1/TAG2) | 물리 서버의 `/etc/tag1`(모델)·`/etc/tag2`(자산 ID)를 프롬프트에 표시하는 강사 개인 컨벤션. 소개만 하고 실습하지 않는다. |
| OIDC | OpenID Connect. OAuth 2.0 위의 로그인(인증) 표준. |
| OVS (Open vSwitch) | 소프트웨어 가상 스위치. → [부록 H.3](H-pve.md#h-3-layout) |
| PATH | 명령 이름을 찾을 디렉터리 목록인 환경변수. 왼쪽부터 찾으며 리눅스는 현재 디렉터리를 뒤지지 않는다. → [Part 4 §4.0](04-container.md) |
| PCI pass-through | 호스트의 PCI 장치를 VM에 직접 넘겨주는 방식. |
| policy (firewalld) | 한 zone에서 다른 zone으로 가는 트래픽에 적용하는 규칙 묶음. → [Part 2 §2.4](02-firewall.md#2-4-firewalld-docker) |
| ProxyJump | `ssh -J`. 중간 서버(bastion)를 경유해 대상에 접속한다. bastion sshd의 `AllowTcpForwarding`이 허용돼 있어야 한다. |
| publish (Docker) | `-p` 옵션으로 컨테이너 포트를 호스트 포트에 연결해 외부에 공개하는 것. → [Part 2 §2.2](02-firewall.md#2-2-docker-bypass) |
| PUEL | VirtualBox Extension Pack의 라이선스. 개인·평가용만 무료, 기관은 유료. → [부록 B](B-licenses.md) |
| RADIUS | 네트워크 접속 인증을 중앙에서 처리하는 프로토콜과 그 서버. 802.1X가 뒤에서 쓴다. |
| RFC 5737 | 문서·예시 전용 IPv4 대역 셋(`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`). 인터넷에서 라우팅되지 않는다. 이 책의 예시 주소. → [들어가며](00-intro.md) |
| RRD (Round Robin Database) | 정해진 크기 안에서 시계열 값을 순환하며 저장하는 형식. PVE의 그래프 데이터가 이 형식이다. |
| SADP | Secure Application Deployment Profile. 앱을 하나하나 검사하는 대신 앱이 올라가는 자리를 입구·출구·빌드·기록의 네 지점에서 통제하자는 지침. 저자가 2026년 제안(<https://msjang.github.io/ostday26-sadp/>). → [Part 7 §7.5](07-ai-governance.md) |
| SNAT | 출발지 주소 변환. 내부 대역이 외부로 나갈 때(masquerade) 쓴다. |
| SR-IOV | Single Root I/O Virtualization. 물리 NIC 하나(PF)를 여러 가상 기능(VF)으로 나눠 VM에 직접 넘기는 기능. → [부록 H.7.1](H-pve.md#h-7-1-sriov) |
| SSPL | MongoDB 등이 쓴 Server Side Public License. 서비스 제공 시 소스 공개 의무, OSI 비승인. |
| Stateful Firewall | 연결 상태를 추적해, 허용한 연결의 응답 패킷을 자동으로 통과시키는 방화벽. |
| sysprep | 이미지 복제 전 머신 정체성(machine-id·호스트키·로그)을 지우는 작업. |
| tap 인터페이스 | VM의 가상 NIC를 호스트 쪽에서 다루는 가상 인터페이스. 하이퍼바이저 방화벽이 여기에 걸린다. → [부록 I.3](I-access-control.md#i-3-hypervisor) |
| TLS 핸드셰이크 | 암호화 통신을 시작하기 전에 양쪽이 인증서와 키 정보를 주고받는 절차. |
| unix socket (유닉스 도메인 소켓) | `IP:포트` 대신 파일 경로(`/var/run/docker.sock` 등)로 여는 소켓. 같은 컴퓨터 안 프로그램끼리의 대화(IPC·RPC)는 대부분 이것이며, 파일 권한이 곧 접근 통제다. → [Part 4 §4.0](04-container.md) |
| V2P | Virtual to Physical. 가상 이미지를 물리 서버로 복원·이식하는 것. |
| veth | 양 끝이 이어진 가상 이더넷 쌍. 컨테이너의 네트워크 네임스페이스를 호스트 브리지에 잇는다. |
| X.509 | 인증서의 국제 표준 형식. HTTPS 서버 인증서, mTLS 클라이언트 인증서, 공동(옛 공인)인증서가 모두 이 형식이다. → [Part 5 §5.1](05-trust.md) |
| zone (firewalld) | 네트워크를 신뢰 수준별로 나눈 구역. 출발지나 인터페이스로 트래픽을 구역에 배정한다. → [Part 2 §2.1](02-firewall.md#2-1-ufw-firewalld) |
