# Part 1. 이미지 — 서버를 "만들어 두고" 쓴다

> 담당 랩 [실습①](labs/lab-01.md)·[실습②](labs/lab-02.md). 랩에서 손으로 만든 뒤, 여기서 **왜 그렇게 하는지**를 본다.

이 파트를 관통하는 생애주기는 한 줄입니다.

```
받기 → 탐색 → 패치 → 정체성 리셋 → 배포 → 복구
```

각 절은 이 고리의 한 칸을 설명합니다. 마지막 두 절(1.8·1.9)은 "이미지 바깥"의 재현성과 운반을 다룹니다.

## 1.1 왜 ISO를 안 까는가 — 골든 이미지와 선언형 운영

**멘탈모델:** 서버는 *설치하는 것*이 아니라 *만들어 둔 파일을 붓는 것*이다.

ISO로 설치하면 마법사에서 누른 선택(파티션·패키지·계정)이 사람의 손과 기억에 남습니다. 같은 서버를 두 번 만들면 두 번 다르고, 왜 다른지 기록도 없습니다. 반대로 **골든 이미지**는 "이 상태가 정답"이라는 결과물 자체가 파일 하나입니다. 바꾸고 싶으면 파일을 고치고, 다시 만들고 싶으면 파일을 붓습니다.

여기서 운영 방식이 바뀝니다. 절차(명령형: "이걸 깔고 저걸 눌러라")가 아니라 **원하는 상태(선언형)** 를 정의하고, 그 상태를 반복해서 찍어냅니다. [실습①](labs/lab-01.md)의 `virt-customize` 한 덩어리가 곧 그 선언입니다. 스크립트는 git에 두고, 비밀(비번)만 밖에 둡니다.

이점은 속도보다 **감사 가능성**입니다. "이 서버에 무엇이 들어 있나"가 사람의 기억이 아니라 파일과 스크립트에 있으면, 사고가 났을 때 되돌릴 기준점이 있습니다. 복구([1.5](#15-배포의-세-길과-복구--pve-clone--v2p--clonezilla))가 분 단위로 가능한 것도 같은 이유입니다.

더 깊이: [cloud-init 문서](https://cloudinit.readthedocs.io/) (첫 부팅 때 선언을 적용하는 표준 방식), [Ubuntu cloud images](https://cloud-images.ubuntu.com/).

## 1.2 클라우드 이미지 해부 — qcow2/raw, OVA/OVF, thin vs thick

**멘탈모델:** 디스크 이미지는 "포장 방식이 다른 디스크 한 장"이다. 내용이 같아도 포장이 다르면 크기·속도·기능이 다르다.

현장에서 마주치는 형식은 넷입니다.

- **raw:** 디스크 바이트 그대로. 가장 단순하고 빠르며, `dd`로 물리 디스크에 바로 쓸 수 있습니다. 대신 *용량 전체*를 차지합니다(thick).
- **qcow2:** QEMU의 형식. 쓴 만큼만 커지는 **thin**이고 스냅샷·압축·백업체인을 지원합니다. 클라우드 이미지가 이 형식으로 배포되는 이유입니다.
- **VMDK/VHD(X):** VMware·Hyper-V의 형식. 변환으로 오갑니다.
- **OVA/OVF:** 디스크 *하나*가 아니라 "VM 정의서(OVF, XML) + 디스크들"을 묶은 포장입니다. OVA는 그걸 tar로 묶은 것. 하이퍼바이저 사이를 옮길 때 쓰고, 안의 디스크는 VMDK인 경우가 많습니다.

thin이 항상 이득은 아닙니다. 쓰기 때마다 공간을 늘리므로 단편화·성능 비용이 있고, 호스트 저장소를 *초과 약정(over-provisioning)* 하면 어느 날 갑자기 꽉 차서 VM 전체가 멈춥니다. 반대로 thick은 예측 가능하지만 낭비입니다. 배포용 골든 이미지는 thin(작아서 옮기기 쉬움), 운영 디스크는 용량 계획과 함께 고르는 것이 보통입니다.

```bash
qemu-img info ⟨image.img⟩                       # 형식, 가상 크기 vs 실제 크기
qemu-img convert -O raw ⟨in.qcow2⟩ ⟨out.raw⟩     # 형식 변환 (V2P의 핵심 명령이기도 하다)
```

`info`의 *virtual size*(게스트가 보는 크기)와 *disk size*(파일이 실제 차지하는 크기)가 다르면 thin입니다.

더 깊이: [QEMU qcow2 스펙](https://www.qemu.org/docs/master/interop/qcow2.html), [qemu-img 매뉴얼](https://www.qemu.org/docs/master/tools/qemu-img.html), [DMTF OVF 표준](https://www.dmtf.org/standards/ovf).

## 1.3 끄고 고치기 — libguestfs와 오프라인이 안전한 이유

**멘탈모델:** 이미지는 *켜야만 보는 블랙박스가 아니다.* 파일시스템이 들어 있는 파일일 뿐이고, 꺼진 채로 열어 읽고 고칠 수 있다.

[libguestfs](https://libguestfs.org/) 도구 모음은 이미지를 작은 임시 VM(어플라이언스)으로 열어 그 안의 파일시스템을 대신 만져 줍니다. 읽기용은 이렇습니다.

```bash
virt-inspector -a ⟨image.img⟩ | head -40   # OS 종류·버전·설치 앱
virt-ls  -a ⟨image.img⟩ /etc                # 디렉터리 목록
virt-cat -a ⟨image.img⟩ /etc/os-release     # 파일 한 개 뽑기
```

쓰기용은 `virt-customize`(계정·패키지·파일·첫 부팅 명령)입니다. 디스크를 *진짜 폴더처럼* 마운트하고 싶으면 `qemu-nbd`로 이미지를 블록 장치(`/dev/nbd0`)로 노출한 뒤 `mount`합니다. 두 방식은 용도가 다릅니다. libguestfs는 *안전하고 스크립트하기 좋고*, nbd 마운트는 *호스트 커널이 직접 만지므로 자유롭지만 실수에 취약*합니다. 마운트했으면 반드시 `umount` 후 `--disconnect`로 끝내야 이미지가 깨지지 않습니다.

**왜 꺼진 채로 하나.** 켜는 순간 이미지는 *다른 물건*이 됩니다. 서비스가 떠서 로그와 상태 파일이 쌓이고, 네트워크에 붙어 외부와 통신하고, DHCP 임대·SSH 키·`machine-id`가 그 인스턴스 고유의 값으로 굳어집니다. 골든 이미지를 한 번 켜 보는 순간 "깨끗한 원본"이 아니게 되는 겁니다. 오프라인이면 이 중 아무것도 일어나지 않으므로, 패치는 **재현 가능하고 부작용이 없습니다.**

같은 이유로 *켜진* 이미지를 오프라인 도구로 건드리면 안 됩니다(게스트가 쓰는 중인 파일시스템과 충돌해 손상). [실습①](labs/lab-01.md)의 경고 "꺼진 상태에서만"은 그 뜻입니다.

보안 디테일 하나. 비번을 `--root-password password:평문`으로 쓰면 셸 히스토리와 `ps`에 남습니다. `--root-password file:pass.txt`로 파일에서 읽히고(`chmod 600`), 끝나면 `shred -u`로 지웁니다. 그 파일은 저장소에 커밋하지 않습니다.

더 깊이: [libguestfs 문서](https://libguestfs.org/guestfs.3.html), [virt-customize](https://libguestfs.org/virt-customize.1.html), [qemu-nbd](https://www.qemu.org/docs/master/tools/qemu-nbd.html).

## 1.4 정체성 리셋 — virt-sysprep, machine-id, SSH hostkey, SELinux .autorelabel

**멘탈모델:** 이미지를 복제할 때 *복제되면 안 되는 것*은 데이터가 아니라 **정체성**이다.

한 서버를 꾸린 뒤 그대로 100대에 붓는다고 합시다. 100대가 모두 같은 `machine-id`, 같은 SSH 호스트 키, 같은 로그를 갖습니다. 그 결과가 생각보다 고약합니다.

- **machine-id 중복:** systemd와 DHCP 일부가 이를 식별자로 씁니다. 같은 값이면 같은 IP를 받거나 로그가 뒤섞입니다.
- **SSH hostkey 중복:** 한 대의 개인키가 새면 100대 전부 사칭당합니다. 클라이언트는 "서버가 같다"고 신뢰합니다.
- **로그·셸 히스토리·임시 파일:** 원본 작업자의 흔적이 모든 복제본에 배포됩니다.

`virt-sysprep`이 이를 오프라인으로 지웁니다. 실행하면 `Performing "machine-id"…` 같은 줄이 수십 개 나옵니다. 지워진 `machine-id`와 호스트 키는 **첫 부팅 때 새로 생성**됩니다. 그래서 [실습②](labs/lab-02.md)에서 부은 뒤 `cat /etc/machine-id`가 클론과 겹치지 않는 새 값으로 나오는 겁니다.

주의할 순서가 있습니다. **패치(customize) 다음에 sysprep**, 그리고 sysprep 이후엔 *그 이미지를 켜지 않습니다.* 켜면 정체성이 다시 생기므로 리셋이 무효가 됩니다.

Rocky/RHEL 계열에는 한 가지가 더 있습니다. 오프라인으로 파일을 쓰면 SELinux 보안 라벨이 없거나 틀립니다. 그대로 부팅하면 서비스가 거부당하거나 로그인이 막힙니다. 그래서 `--touch /.autorelabel`로 **다음 부팅 때 전체 재라벨**을 예약합니다. 첫 부팅이 평소보다 길어지는 건 정상입니다. 루트 비번 복구([1.6](#16-열쇠를-잃었을-때--루트-비번-복구와-그-함의))에서도 같은 이유로 등장합니다.

더 깊이: [virt-sysprep](https://libguestfs.org/virt-sysprep.1.html), [machine-id(5)](https://www.freedesktop.org/software/systemd/man/latest/machine-id.html), [RHEL SELinux 문서](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/index).

## 1.5 배포의 세 길과 복구 — PVE Clone / V2P / Clonezilla

**멘탈모델:** 되돌리는 세 가지 방법은 *무엇을 뜨고, 어디서 누르는가*가 다르다.

| 길 | 무엇을 | 어디서 | 쓰임 |
| --- | --- | --- | --- |
| **PVE Clone** | VM의 *현재 상태* | 하이퍼바이저 웹 | 가상 세계의 undo 버튼 |
| **V2P** | *골든 이미지*를 디스크에 붓기 | 라이브 USB + `qemu-img convert` | 새 서버·재설치를 "준비된 상태"로 |
| **Clonezilla** | *지금 이 서버*의 디스크 | 라이브 USB (어디서나) | 물리 서버의 백업·복원 |

**Clone**은 가장 쉽습니다. 뭔가 위험한 일을 하기 전에 먼저 누르는 안전망입니다. 하지만 *하이퍼바이저가 있어야* 눌립니다.

**V2P**는 이름 그대로 가상(Virtual) 이미지를 물리(Physical) 디스크에 옮깁니다. 핵심은 `qemu-img convert -O raw ⟨이미지⟩ /dev/⟨디스크⟩` 한 줄. qcow2를 raw로 풀어 블록 장치에 직접 씁니다. [1.4](#14-정체성-리셋--virt-sysprep-machine-id-ssh-hostkey-selinux-autorelabel)에서 정체성을 리셋한 이미지라면, 같은 이미지를 몇 대에 부어도 각자 다른 서버로 깨어납니다. 설치 마법사를 한 번도 거치지 않습니다.

**Clonezilla**는 물리 서버에는 Clone 버튼이 없다는 문제의 답입니다. 라이브 USB로 부팅하면 어떤 장비에서든 그 버튼이 생깁니다. V2P와 달리 *골든*이 아니라 **지금 상태**(설정·데이터 포함)를 통째로 뜨고, **쓰인 블록만 압축**하므로 디스크 용량보다 훨씬 작습니다. 그래서 "이미지를 만든다"(V2P 재료)와 "상태를 지킨다"(Clonezilla)를 구분해 둬야 합니다.

가장 위험한 순간은 `convert`의 **대상 디바이스를 고르는 때**입니다. `/dev/sda`와 `/dev/sdb`를 바꿔 입력하면 반대쪽이 덮입니다. `lsblk`로 크기·모델·마운트 여부를 눈으로 확인하는 습관이 안전의 전부입니다.

```bash
lsblk -o NAME,SIZE,MODEL,MOUNTPOINT    # 쓰기 전에 반드시. 대상이 정말 빈 디스크인가?
```

복구 관점에서 요약하면, 깨끗한 재설치는 V2P, 이 서버 그대로 복원은 Clonezilla, VM이면 Clone입니다. 셋을 모두 갖춘 곳이 "디스크를 날려도 분 단위로 되살리는" 곳입니다.

더 깊이: [Clonezilla 문서](https://clonezilla.org/), [Proxmox VE: Clone](https://pve.proxmox.com/wiki/Qemu/KVM_Virtual_Machines), [qemu-img convert](https://www.qemu.org/docs/master/tools/qemu-img.html).

## 1.6 열쇠를 잃었을 때 — 루트 비번 복구와 그 함의

**멘탈모델:** 디스크를 손에 쥔 순간 로그인 화면은 장식이다. **콘솔이나 부팅 매체에 접근할 수 있으면 곧 루트다.**

[실습②ⓕ](labs/lab-02.md)의 두 방법은 같은 원리입니다. 운영체제가 *깨어 있는 동안*에만 비밀번호가 의미를 가지므로, 깨우기 전에 파일(`/etc/shadow`)을 직접 고치면 됩니다. 방법 A는 GRUB에서 `init=/bin/bash`(RHEL은 `rd.break`)로 인증을 건너뛰고, 방법 B는 라이브 USB로 루트 파티션을 마운트해 `chroot`로 `passwd`를 실행합니다. 앞의 [1.3](#13-끄고-고치기--libguestfs와-오프라인이-안전한-이유)에서 본 `virt-customize --root-password`와 뿌리가 같습니다. 오프라인으로 디스크를 만지는 것이니까요.

이게 *복구 요령*이면서 동시에 *공격 요령*이라는 점이 핵심입니다. 이 방법을 쓸 수 있는 사람은 누구든 루트가 됩니다. 그래서 다음은 "있으면 좋은" 항목이 아니라 **필수**입니다.

- **BIOS/UEFI 암호 + 부팅 순서 잠금:** USB 부팅을 막습니다.
- **GRUB 암호:** `e` 키 편집(방법 A)을 막습니다.
- **IPMI/BMC 보안:** 원격 콘솔도 콘솔입니다. 기본 계정·약한 비번은 곧 루트입니다. 관리망 격리 포함.
- **디스크 암호화(LUKS):** 방법 B를 끝냅니다. 패스프레이즈 없이는 파티션이 마운트되지 않습니다.
- **물리 접근 통제와 [bastion](06-access.md)(Part 6 접근):** 문 잠금, 그리고 접근 경로를 한 곳으로.

방어는 겹으로 쌓습니다. LUKS 하나가 가장 단단하지만, 부팅 때마다 패스프레이즈를 입력해야 하므로 원격 서버에서는 운영 부담이 생깁니다. 그 비용을 의식하고 선택하는 것이 "안전한 운영"입니다.

더 깊이: [GRUB 보안 매뉴얼](https://www.gnu.org/software/grub/manual/grub/html_node/Security.html), [cryptsetup/LUKS](https://gitlab.com/cryptsetup/cryptsetup), [Red Hat: 루트 비번 재설정](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_basic_system_settings/index).

## 1.7 하드웨어 신원 — inxi·IPMI·서비스태그·자산 스티커

**멘탈모델:** 이미지가 "무엇을 설치했나"의 신원이라면, 하드웨어 신원은 "*지금 내가 어느 장비 앞에 있나*"의 신원이다.

데이터센터에는 같은 모양의 서버가 줄지어 있습니다. 엉뚱한 장비에 `reboot`하거나 디스크를 `convert`하는 사고는 대개 **내가 어느 장비에 있는지 몰라서** 납니다. 그래서 장비는 여러 층의 이름표를 가져야 하고, 그 이름표들이 서로 일치하는지 확인하는 것이 인수 절차입니다. 사내 위키 「서버 납품 가이드」의 인벤토리 절차를 요약하면 이렇습니다.

```bash
sudo inxi -F                      # 모델·시리얼·CPU·메모리·디스크를 한 번에
sudo ipmitool lan print 1         # BMC의 IP, MAC 주소 (관리망 신원)
```

- **시리얼/서비스태그:** 제조사가 부여한 고유값. `inxi -F`에서 나오고, 기기 앞뒤 스티커·서비스태그 라벨과 *눈으로 대조*합니다. 제조사 보증·지원 요청의 키이기도 합니다.
- **IPMI MAC:** 관리 포트(BMC)의 MAC. DHCP 예약이나 관리망 ACL의 키가 됩니다. 본체 NIC MAC과 다른 값입니다.
- **자산 스티커:** 기관의 자산 번호. 사람이 보는 층입니다.

이 값들을 *한곳에 모은 기록*(인벤토리 표)이 없으면 장애 때 "이 시리얼이 몇 번 랙의 어느 장비지?"를 몸으로 찾아야 합니다. 그리고 그 기록을 **서버 안에도** 넣어 두면, 접속한 순간에도 내가 어디 있는지 알 수 있습니다. [실습①](labs/lab-01.md)에서 `/etc/tag1`(모델)과 `/etc/tag2`(자산 ID)를 이미지에 심고, ohmyzsh 프롬프트가 `[TAG1 … | TAG2 …]` 형태로 보여주는 것이 바로 그 연결입니다. 이미지 쪽 신원(1.4)과 하드웨어 쪽 신원이 프롬프트 한 줄에서 만납니다.

주의: 이미지는 *여러 장비에 붓는* 것이므로, 태그를 이미지 안에 구워 넣으면 모든 장비가 같은 태그를 갖게 됩니다. 장비별 값은 **붓고 난 뒤**(또는 첫 부팅 시) 채우거나, 태그는 "골든의 모델 계열"까지만 넣는 식으로 역할을 나눠야 합니다.

더 깊이: [inxi 문서](https://smxi.org/docs/inxi.htm), [ipmitool 매뉴얼](https://linux.die.net/man/1/ipmitool).

## 1.8 재현가능 환경 — 패키지 관리자 지형과 marimo

**멘탈모델:** "내 노트북에서는 되는데"의 해결책은 *환경을 더 많이 기억하는 것*이 아니라 **환경을 파일로 고정하는 것**이다.

패키지 관리자는 고정하는 범위가 점점 넓어지는 계단입니다.

1. **apt/dnf (OS 패키지):** 시스템 전체. 배포판이 관리하므로 안정적이지만 버전이 느리고, 프로젝트마다 다른 버전이 필요하면 충돌합니다.
2. **pip / [uv](https://docs.astral.sh/uv/) (파이썬 패키지):** 프로젝트별 가상환경. `pip freeze`나 uv의 lock 파일로 버전을 고정합니다. 파이썬만 고정하고 C 라이브러리·CUDA 같은 *비파이썬 의존성*은 못 잡습니다.
3. **conda 계열:** 파이썬 *바깥*(컴파일된 라이브러리)까지 환경 단위로 고정합니다. 과학 계산에서 사랑받는 이유입니다.
4. **컨테이너:** OS 사용자 공간까지 통째로 고정합니다. **최종 재현**의 단계입니다([Part 4](04-container.md)).

conda에는 함정이 하나 있습니다. **Anaconda 배포판의 기본 채널(`defaults`)은 일정 규모 이상의 조직에는 상용 라이선스가 요구됩니다.** 연구실이 무심코 Anaconda를 깔아 쓰다가 라이선스 위반이 되는 전형적인 섀도우 IT입니다. 대안은 **miniforge**(커뮤니티 `conda-forge` 채널이 기본, 무료)입니다. 같은 `conda` 명령을 쓰되 채널만 안전하게 바뀝니다. 즉 **anaconda ▶ miniforge**입니다. 이 책의 큰 그림("라이선스를 읽고")이 패키지 관리자 선택 하나에서도 적용됩니다.

노트북은 **[marimo](https://marimo.io/)** 를 권합니다. 차이를 한 표로 보면:

| | Jupyter (`.ipynb`) | marimo (`.py`) |
| --- | --- | --- |
| 파일 | JSON(출력 이미지까지 포함) | 순수 파이썬 |
| git diff | 읽기 어렵고 충돌 잦음 | 일반 코드처럼 읽힘 |
| 상태 | 셀 실행 순서에 따른 **숨은 상태** | 셀 간 의존을 분석해 **반응형**(값이 바뀌면 연결된 셀이 다시 실행) |
| 재현 | "위에서부터 다시 실행"해야 확인 | 항상 현재 코드와 결과가 일치 |

핵심은 *숨은 상태*입니다. Jupyter에서는 지운 셀의 변수가 메모리에 남아 결과가 "우연히" 맞을 수 있고, 그 노트북은 다른 사람 컴퓨터에서 재현되지 않습니다. marimo는 이 가능성을 구조적으로 차단하며, `.py`라서 코드 리뷰와 git 이력에 그대로 올라갑니다. 의존성은 노트북 상단에 선언하고(`uv`와 궁합이 좋습니다) 실행하면 재현 환경이 한 파일에 담깁니다.

더 깊이: [conda-forge](https://conda-forge.org/), [miniforge](https://github.com/conda-forge/miniforge), [Anaconda 서비스 약관](https://www.anaconda.com/legal), [marimo 문서](https://docs.marimo.io/).

## 1.9 Ventoy 한 자루 — 부팅도구+이미지 한 USB

**멘탈모델:** USB 하나가 *부팅도구(ISO)* 와 *운반할 짐(이미지)* 을 둘 다 싣는다. ISO는 복사만 하면 메뉴에 뜬다.

보통의 부팅 USB는 ISO 하나를 통째로 써서 구우므로, 도구마다 USB가 하나씩 필요합니다. **[Ventoy](https://www.ventoy.net/)** 는 USB에 부트로더를 *한 번만* 설치하고, 그 위에 ISO 파일을 그냥 복사해 둡니다. 부팅하면 파일 목록이 메뉴로 나옵니다. 현장 가방에는 이것 하나면 됩니다. [실습②](labs/lab-02.md)ⓐ가 이 구성입니다.

- **부팅도구:** GParted(라이브 터미널, V2P 실행 환경), Clonezilla(백업·복원), 아무 라이브 ISO(루트 복구).
- **짐(페이로드):** [실습①](labs/lab-01.md)에서 만든 패치된 골든 이미지.

설치 때 `-r ⟨MB⟩`로 뒤쪽에 *예비 데이터 파티션*을 남길 수 있습니다. Clonezilla가 백업을 쓸 저장소로 쓰려면 포맷이 중요합니다. exFAT은 호환 문제가 있어 ext4/ntfs/fat을 권하고, 이 때문에 USB 빌드 쪽에서 `DATA_FS=ext4` 같은 선택지를 둡니다.

이 책의 USB 이미지는 `make-usb-ventoy.sh`로 만듭니다. 흐름만 보면 이렇습니다.

1. 빈 **thin qcow2**를 만든다(`qemu-img create`, 쓴 만큼만 커지므로 몇 GB짜리 USB도 파일은 작다).
2. 그것을 블록 장치(`qemu-nbd`)로 노출하고 Ventoy를 설치한다.
3. ISO들과 이미지를 데이터 파티션에 복사하고 분리한다.

앞 절들의 도구가 한 스크립트에 모두 모입니다. thin qcow2([1.2](#12-클라우드-이미지-해부--qcow2raw-ovaovf-thin-vs-thick)), nbd([1.3](#13-끄고-고치기--libguestfs와-오프라인이-안전한-이유))가 그것입니다. 수강생에게는 이 qcow2를 VM에 USB 디스크로 붙여 "꽂는 느낌"을 연습하게 하고, 현장에서는 같은 구성을 실물 USB에 `dd`하거나 Ventoy를 직접 설치하면 됩니다.

한 가지 운영 주의. 이 USB는 *루트 복구 도구*([1.6](#16-열쇠를-잃었을-때--루트-비번-복구와-그-함의))이자 골든 이미지(내부 설정이 든)를 담은 물건입니다. 분실하면 곧 사고이므로, 이미지에는 비밀을 넣지 않고 USB 자체도 자산처럼 관리합니다.

더 깊이: [Ventoy 문서](https://www.ventoy.net/en/doc_start.html), [GParted Live](https://gparted.org/livecd.php), [Clonezilla Live](https://clonezilla.org/clonezilla-live.php).
