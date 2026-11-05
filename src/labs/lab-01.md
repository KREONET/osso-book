## 실습① 클라우드 이미지 — 받아서, 안 켜고 열어보고, 고친다

🎯 **목표:** 부팅 없이 이미지 내부를 **탐색**하고, 계정·정체성을 **패치**하고, 설정을 **sed로 고치지 않고 조각 파일로 얹는다.**

🖥️ **랩:** 작업 VM(libguestfs·qemu-utils 설치됨). 이미지는 파일 하나로 다룬다.

⌨️ **받기**
```bash
wget ⟨https://⟨mirror.example⟩/.../jammy-server-cloudimg-amd64.img⟩ -O ⟨image.img⟩
```

📦 **클라우드 이미지 공식 저장소** — 최신은 `current`·`latest` 디렉터리에 있다.

| 배포판 | 저장소 (최신) | 파일 |
| --- | --- | --- |
| Ubuntu 26.04 LTS | <https://cloud-images.ubuntu.com/resolute/current/> | `resolute-server-cloudimg-amd64.img` |
| Ubuntu 24.04 LTS | <https://cloud-images.ubuntu.com/noble/current/> | `noble-server-cloudimg-amd64.img` |
| Debian 13 | <https://cloud.debian.org/images/cloud/trixie/latest/> | `debian-13-genericcloud-amd64.qcow2` |
| Rocky 10 | <https://dl.rockylinux.org/pub/rocky/10/images/x86_64/> | `Rocky-10-GenericCloud-Base.latest.x86_64.qcow2` |
| Rocky 9 | <https://dl.rockylinux.org/pub/rocky/9/images/x86_64/> | `Rocky-9-GenericCloud-Base.latest.x86_64.qcow2` |

> 우분투의 `current/`는 보안 업데이트가 반영된 **최신 빌드**다. 버전을 고정하려면 `releases/26.04/`처럼 릴리스 디렉터리를 쓴다. 받은 뒤엔 같은 디렉터리의 체크섬 파일(`SHA256SUMS` 등)로 검증한다. 우분투 `.img`도 확장자만 그렇고 실제 형식은 qcow2다(`qemu-img info`로 확인). 데비안의 `genericcloud`는 가상머신용이라 물리 하드웨어 드라이버가 빠져 있다. 실물 서버에 부을(②의 V2P) 이미지면 `generic`을 받는다.

⌨️ **탐색 — 안 켜고 안을 들여다본다(읽기)**
```bash
# OS·설치앱 목록
virt-inspector -a ⟨image.img⟩ | head -40
# 디렉터리 보기 (부팅 없이)
virt-ls  -a ⟨image.img⟩ /etc | head
# 파일 내용 뽑기
virt-cat -a ⟨image.img⟩ /etc/os-release
# 진짜 파일시스템으로 마운트해서 보기 (qemu-nbd)
sudo modprobe nbd max_part=8
sudo qemu-nbd --connect=/dev/nbd0 ⟨image.img⟩
# 디스크 → 파티션 → 파일시스템, 세 겹을 본다. 루트는 몇 번이고 FS는 무엇인가?
lsblk -f /dev/nbd0
# 루트 파티션을 폴더처럼 (우분투·데비안 p1, Rocky p4)
sudo mount /dev/nbd0p⟨1⟩ /mnt && ls /mnt/etc
sudo umount /mnt; sudo qemu-nbd --disconnect /dev/nbd0
```

> 파티션 **번호**는 디스크 안의 **위치**가 아니다. 우분투 이미지는 루트가 1번이지만 디스크 맨 뒤에 있고, 앞쪽은 14·15·16번(BIOS·EFI·`/boot`)이 차지한다. Rocky는 4번이 루트이고 FS가 XFS다. 그래서 번호를 외우지 말고 `lsblk -f`로 확인한다([Part 1 §1.2](../01-image.md)).

⌨️ **키우기 — 이미지는 일부러 작게 온다**
```bash
# 가상 크기를 본다 (우분투 3.5 GiB)
qemu-img info ⟨image.img⟩ | grep 'virtual size'
# 디스크 겹만 20 GiB로
qemu-img resize ⟨image.img⟩ 20G
# 예측: 루트 파일시스템도 20 GiB가 됐을까?
virt-df -h -a ⟨image.img⟩
```

👀 `virt-df`의 루트 크기는 **그대로**다. 디스크만 커졌고 안쪽의 파티션과 파일시스템은 아직 옛 크기다. 키우는 방법은 둘이다. cloud-init이 돌면 **첫 부팅 때** 파티션과 파일시스템을 디스크 끝까지 알아서 늘린다. 안 돌면 부팅한 뒤 손으로 세 단계(`growpart` → `resize2fs`/`xfs_growfs`)를 한다. ②에서 `df -h /`로 어느 쪽이었는지 확인한다.

⌨️ **패치 — 계정·패키지·운영환경·정체성을 심는다(쓰기)**
```bash
# 비번은 명령줄에 직접 쓰지 않는다 → 파일로 (셸 히스토리·ps 노출 방지)
# 방법 1: 직접 정하기
printf '%s' '⟨labpass⟩' > pass.txt
# 방법 2: 무작위로 만들기 — 쓰려면 위 printf 대신 아래 두 줄의 # 을 지운다
# openssl rand -base64 18 > pass.txt
# cat pass.txt
chmod 600 pass.txt
```

`openssl rand` 출력 예 (매번 다르다):
```text
ntQOkXgPRDZGo84g9DCMM7C8
```

> 18바이트 무작위 값을 base64로 바꾸면 24글자가 된다(18이 3의 배수라 끝에 `=`가 붙지 않는다). 무작위 비번이면 아래 `shred`로 파일을 지우기 **전에** 비밀번호 관리자 같은 안전한 곳에 옮겨 둔다. 안 그러면 ②에서 부팅한 뒤 로그인할 방법이 없다.

```bash
virt-customize -a ⟨image.img⟩ \
  --root-password file:pass.txt \
  --run-command 'useradd -m -s /bin/bash ⟨labuser⟩' \
  --password ⟨labuser⟩:file:pass.txt \
  --install htop,ncdu \
  --run-command 'systemctl disable snapd.service' \
  --firstboot-command \
    'curl -s https://raw.githubusercontent.com/KREONET/ohmyzsh/main/install.sh | sh -s -'
# 끝나면 비번 파일 폐기(흔적 제거)
shred -u pass.txt
# 정체성 리셋(machine-id·hostkey·로그)
virt-sysprep -a ⟨image.img⟩
# ⚠️ Rocky/RHEL 계열이면 다음 부팅 때 SELinux 재라벨:
# virt-customize -a ⟨rocky.qcow2⟩ --touch /.autorelabel
```

> 비번은 `file:pass.txt` 형태로 — `password:⟨평문⟩`은 셸 히스토리·`ps`·스크립트에 그대로 남는다. 파일로 받고 끝나면 `shred -u`로 지운다(저장소에 **커밋 금지**).

> 오프라인 편집 중엔 네트워크가 없다 → ohmyzsh는 `--firstboot-command`로 **첫 부팅 때** 설치되게 심는다(egress 허용 필요 — Day1 ⑥과 연결).

⌨️ **설정은 "고치는 것"이 아니라 "얹는 것" — 드롭인(.d)을 직접 발견한다**

리눅스 설정은 큰 파일 하나가 아니라, 큰 파일이 작은 조각들을 **불러오는(include)** 구조인 경우가 많다. 먼저 **직접 확인**하자.
```bash
# 주석 걷어내고 '실제 설정만' 본다
grep -v '#' /etc/ssh/sshd_config | uniq
```
맨 위(또는 아래)에 이런 줄이 보일 것이다.
```
Include /etc/ssh/sshd_config.d/*.conf
```

> ⚠️ 이 구조는 **배포판·버전마다 다르다.** 어떤 이미지엔 이 `Include`가 없을 수도, 경로가 다를 수도 있다. "직접 열어 확인"이 먼저다(이 랩이 외우기보다 *확인하는 습관*을 들이는 이유).

그렇다면 원본(`sshd_config`)을 `sed`로 고치지 말고, `*.d/`에 **조각 파일 하나를 얹는다.** `sed`는 "그 줄이 있나? 주석 처리됐나? 또 추가되진 않나?"를 매번 걱정해야 하지만, **복사는 그냥 파일을 놓는 일**이라 멱등(여러 번 해도 같은 결과)하다.
```bash
# 얹을 조각을 만든다 — 주석은 반드시 '별도 줄'로 (아래 ⚠️ 참고)
cat > 60-lab.conf <<'EOF'
# 랩 전용 보안 기본값
PasswordAuthentication no
PermitRootLogin prohibit-password
EOF

# 이미지 안 .d 디렉터리에 '복사해 넣는다' (sed 아님!)
virt-copy-in -a ⟨image.img⟩ 60-lab.conf /etc/ssh/sshd_config.d/
#  동등:  virt-customize -a ⟨image.img⟩ --copy-in 60-lab.conf:/etc/ssh/sshd_config.d/
#  경로·권한까지 지정:  --upload 60-lab.conf:/etc/ssh/sshd_config.d/60-lab.conf
```

> ⚠️ **주석은 설정 줄 끝에 붙이지 말고 별도 줄로.** `PasswordAuthentication no  # 랩용` 처럼 쓰면 방금 본 `grep -v '#'`가 **설정 줄 통째로** 걸러 버린다. 게다가 많은 파서는 인라인 `#`을 주석으로 안 보고 **값의 일부로** 읽는다. 그래서 관례가 "주석은 윗줄에".

> ⚠️ **RHEL 계열에서 포트를 바꾸면(예: `Port 2222`) SELinux가 막는다.** 설정은 맞는데 서비스가 안 뜬다 → 흔한 유혹이 "SELinux 끄기"다. **끄지 말 것.** 포트에 라벨을 알려 주면 된다: `sudo semanage port -a -t ssh_port_t -p tcp 2222`. 왜 끄면 안 되는지, 로그(`ausearch`) 읽고 규칙 만드는 법은 [Part 1 §1.4](../01-image.md)에서.

⌨️ **생각해보기:** 아래도 각자 `*.d/`가 있다. "원본을 고칠까, 조각을 얹을까?" — 답은 거의 늘 **얹기**다. 어느 걸 `virt-copy-in`으로 넣어 보겠는가?
`/etc/sudoers.d/` · `/etc/rsyslog.d/`(로그 라우팅) · `/etc/cron.d/` · `/etc/profile.d/`(로그인 환경·프롬프트) · `/etc/sysctl.d/`(커널 파라미터) · `/etc/systemd/system/⟨서비스⟩.service.d/`(유닛 오버라이드) · `/etc/apt/apt.conf.d/`·`sources.list.d/` · `/etc/yum.repos.d/` · `/etc/NetworkManager/conf.d/` · `/etc/modprobe.d/` · `/etc/logrotate.d/`

> **"있는 것"과 "켜진 것"을 링크로 나누기도 한다.** 아파치는 `sites-available/`에 설정을 모아 두고 `sites-enabled/`로 **`ln -s`** 해서 활성화한다(`a2ensite`가 곧 그 심볼릭 링크, `a2dissite`는 링크 삭제). nginx도 같은 관습이고, `systemctl enable`조차 사실은 `*.wants/`에 유닛을 **링크**하는 일이다. → 설정을 *지우지 않고* 링크만 끊어 끌 수 있다.

👀 **보이는 것:** inspector가 OS·앱을 뱉고, `virt-ls`/`qemu-nbd`로 안의 파일이 그대로 보인다. `grep` 한 줄에 `Include .../sshd_config.d/*.conf`가 뜬다. 조각을 넣은 뒤 `virt-ls -a ⟨image.img⟩ /etc/ssh/sshd_config.d` 하면 방금 얹은 `60-lab.conf`가 목록에 있다. customize는 `Installing packages…`, sysprep은 `Performing "machine-id"…`를 수십 줄 쏟아낸다.

💡 **느낄 것:** 이미지가 **켜야만 보는 블랙박스가 아니다.** 안 켜고 열어보고(virt-ls·qemu-nbd) 고친다(virt-customize). 그리고 리눅스 설정은 **윈도우 레지스트리(하나의 불투명한 바이너리 하이브)와 달리 평문 파일**이다 — `diff`로 비교되고, git에 올라가고, `.d/`에 **조각만 얹어** 바꾸고, 링크로 켜고 끈다. 그래서 프로비저닝의 정답은 "원본을 `sed`로 깎기"가 아니라 "**조각 파일을 복사해 얹기**"다. 원본은 배포판이 주는 대로 두고, 내 결정만 한 조각으로 떼어 놓으면 — 무엇을 왜 바꿨는지가 파일 하나로 남는다([Part 1 §1.3](../01-image.md) 더 깊이).

⚠️ 모든 오프라인 편집은 이미지가 **꺼진 상태**에서만.

---

🔀 **갈림길** — 다음엔 어디로?

- 이미지를 디스크에 붓고, 날려도 되살려 보려면 → [② 배포·복원](lab-02.md)
- 왜 꺼진 채로 고치나, 드롭인은 왜 안전한가 → [Part 1 §1.3](../01-image.md)
- RHEL에서 설정은 맞는데 서비스가 안 뜬다 → [Part 1 §1.4 SELinux](../01-image.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
