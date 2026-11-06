## 실습② 배포·복원 — Ventoy·클론·V2P·Clonezilla·루트 비밀번호 복구 {#lab-02}

🎯 **목표:** 준비한 이미지를 디스크에 기록하고(V2P, 가상 머신 이미지를 물리 디스크로 옮기는 일), 지금 상태를 백업·복원(Clonezilla)하고, 비밀번호를 잃어도 되찾는다. **되돌리기 네 갈래**를 익힌다.

🖥️ **랩:** 작업 VM + `usb-ventoy.qcow2` + (삭제·추가할) 디스크. 모두 PVE(Proxmox VE)에서 진행한다.

⌨️ **ⓐ Ventoy USB 하나에 부팅도구 + 이미지 싣기** (작업 VM)
```bash
# usb-ventoy.qcow2 에 Ventoy 설치
sudo ventoy -i /dev/⟨sdb⟩
#   필요시 -r ⟨4096⟩ 로 뒤에 예비(데이터) 파티션 공간 확보
sudo mount /dev/⟨sdb1⟩ /mnt
# 부팅 ISO들 + 패치 이미지
sudo cp gparted-q.iso clonezilla.iso ⟨image.img⟩ /mnt/ ; sync
```

👀 USB 데이터 파티션에 `gparted-q.iso`와 ①의 패치 이미지가 **같이** 들어감.

💡 **USB 하나가 부팅도구와 페이로드를 둘 다 싣는다.** ISO는 복사만 해도 Ventoy 메뉴에 나타난다.

⌨️ **ⓑ 안전망 — 이 머신 클론(백업)** (PVE 웹)
```
작업 VM 우클릭 > Clone > Full Clone  →  되돌릴 수 있는 백업 확보
```

💡 무언가를 지우기 전에, 되돌리기(undo) 버튼 역할을 할 **클론**부터 만든다.

> 💾 **클론·백업을 빠르게 — thin + discard.** thin 저장소(LVM-thin·ZFS·qcow2)는 쓴 만큼만 차지하고, 클론·백업도 **쓰인 블록만** 복사하니 빠르다. 그런데 게스트 안에서 파일을 지워도 저장소는 그 사실을 모른다. 공간은 회수되지 않고, 지운 데이터까지 계속 복사된다. 그래서 VM 디스크에 **Discard**를 켜고, 게스트가 TRIM으로 "여기 비었다"고 알려 주게 한다. (랩 VM은 미리 켜 두었다 — 확인만.)

```
VM > Hardware > Hard Disk > Edit > ☑ Discard  ☑ SSD emulation
→ VM을 완전히 껐다 켜야 적용된다 (게스트 안 reboot로는 안 됨)
VM > Options > QEMU Guest Agent > ☑ Run guest-trim after a disk move or VM migration
```

```bash
# 게스트 안: 주기적 TRIM 타이머가 켜져 있나 (꺼져 있으면 아래 enable)
systemctl status fstrim.timer
sudo systemctl enable --now fstrim.timer
# 지금 바로 한 번 비우기 — 파일시스템마다 비운 양이 나온다
sudo fstrim -av
```

```bash
# PVE 호스트: 실제로 차지하는 양 (LVM-thin이면 Data% 가 줄어든다)
sudo lvs
```

⌨️ **ⓒ 재난 — 메인 디스크를 제거하고 부팅** (PVE 웹)
```
VM > Hardware > 메인 디스크 선택 > Detach → Remove
→ 부팅  →  "No bootable device" (OS가 없으니 당연)
```

👀 멀쩡하던 서버가 부팅되지 않는다. 디스크가 곧 OS다.

⌨️ **ⓓ 복원 — 빈 디스크 추가 + USB로 부팅해 V2P**
```bash
# PVE: Hardware > Add > Hard Disk(20G), 그리고 usb-ventoy 디스크 붙임 → USB로 부팅
#      Ventoy 메뉴 > GParted-q 선택 → 라이브 터미널:
# 타겟 빈 디스크(⟨sda⟩)·Ventoy(⟨sdb⟩) 확인
lsblk
sudo mount /dev/⟨sdb1⟩ /mnt
sudo qemu-img convert -p -f qcow2 -O raw /mnt/⟨image.img⟩ /dev/⟨sda⟩
# ISO·USB 떼고 → 새 디스크로 부팅
```
부팅한 뒤 **되살아났는지 확인**한다.
```bash
# ①에서 넣은 패치가 그대로 있다
htop --version
# sysprep 덕에 새 값(클론과 안 겹침)
cat /etc/machine-id
# 새 장비엔 새 이름 — ⟨IP끝 3자리⟩-⟨OS⟩-⟨용도⟩
sudo hostnamectl set-hostname ⟨042-u24-web⟩
# 예측: 이미지는 작았고 디스크는 20G다. 루트는 지금 몇 G일까?
lsblk -f
df -h /
```

> 📏 **켜진 채로 디스크 키우기 (PVE).** VM이 켜져 있어도 PVE의 디스크 *Resize*(또는 `qm disk resize ⟨vmid⟩ scsi0 +10G`)는 바로 된다. 하지만 커지는 건 **디스크 겹뿐**이다. 안쪽은 게스트에서 따로 키운다.
>
> ```bash
> # 디스크가 커졌는지, 루트가 몇 번이고 FS가 무엇인지
> lsblk -f
> # 우분투·데비안 (루트 p1, ext4)
> sudo growpart /dev/⟨sda⟩ 1
> sudo resize2fs /dev/⟨sda1⟩
> # Rocky (루트 p4, XFS — 장치가 아니라 마운트 지점을 준다)
> sudo growpart /dev/⟨sda⟩ 4
> sudo xfs_growfs /
> df -h /
> ```
>
> cloud-init이 동작하는 VM이라면 재부팅만으로도 된다(부팅할 때마다 같은 일을 한다). 줄이기는 안 된다고 생각하자([Part 1 §1.2](../01-image.md)).

> 💡 VM이 많아지면 `ubuntu@ubuntu`로는 헷갈린다. **`⟨IP 끝자리 3자리⟩-⟨u24 같은 OS약어⟩-⟨용도⟩`** 처럼 이름에 정보를 넣으면 프롬프트만 보고도 어느 서버인지, apt를 쓰는지 dnf를 쓰는지, 재부팅해도 되는지 알 수 있다. OS 버전·IP는 어차피 `/etc/os-release`·`ip a`로 누구나 보는 정보라 보안 손해는 거의 없고 편의는 크다([Part 1 §1.7](../01-image.md)).

👀 **보이는 것:** 처음엔 부팅 실패 → convert 100% → 디스크·USB를 떼고 부팅하면 **OS가 올라오고**, 내 패치(htop)와 **새 정체성**이 확인됨. 작은 이미지(우분투 3.5G)를 20G 디스크에 기록했다. `df -h /`가 20G 가까이면 cloud-init이 첫 부팅 때 늘린 것이고, 그대로면 cloud-init이 꺼져 있는 것이다(`cloud-init status`로 확인). 이때는 앞의 📏 박스에 나온 세 단계로 직접 키운다. 설치 마법사는 한 번도 거치지 않았다.

💡 **느낄 것:** 클론은 **되돌리기**, V2P는 **빈 디스크에 기록하기**다. 준비된 이미지만 있으면 디스크를 잃어도 몇 분 만에 되살린다. 현장에서는 이 Ventoy USB를 가져가 꽂기만 하면 된다. ①의 오프라인 편집이 여기까지 한 줄기로 이어진다.

⚠️ `qemu-img convert` **대상 `/dev/`를 반드시 확인.** 엉뚱한 디스크를 고르면 디스크 전체가 덮어쓰인다.

⌨️ **ⓔ Clonezilla — 하이퍼바이저 없이 백업·복원(물리식)**
저장소로 쓸 디스크(또는 데이터 파티션)를 하나 붙여두고, Ventoy 메뉴 > **Clonezilla** 선택.
```
[백업 savedisk]
1. device-image → local_dev (저장소 = 백업 디스크 선택)
2. savedisk → 소스 디스크 선택 → 이미지 이름 입력
3. 저장 (사용 중인 블록만 압축 → raw 통짜보다 작고 빠름)

[복원 restoredisk]
1. device-image → local_dev (같은 저장소)
2. restoredisk → 이미지 선택 → 타겟 디스크 선택 → 복원
3. 떼고 부팅 → 이전 상태 그대로 되살아남
```

👀 **보이는 것:** 저장 이미지가 **디스크 용량이 아니라 실제 사용량만큼**(압축). 복원 후 서버가 이전 상태 그대로 부팅.

💡 **느낄 것:** PVE Clone은 **하이퍼바이저가 있어야** 누를 수 있는 버튼이다. 물리 서버에는 그 버튼이 없다. Clonezilla는 라이브 USB로 어디서나 그 버튼 역할을 한다. V2P(골든 이미지, 즉 표준으로 미리 만들어 둔 원본 이미지를 기록하는 일)와 달리 이것은 **지금 이 서버의 상태**를 그대로 복제한다.

⚠️ Clonezilla 저장소 FS는 ext4/ntfs/fat 권장(exFAT 호환 이슈). 데이터 파티션을 저장소로 쓰려면 `DATA_FS=ext4`로 만든 USB 사용.

⌨️ **ⓕ 열쇠를 잃었을 때 — 루트 비밀번호 복구** (같은 USB로, 이번에는 시스템이 아니라 *열쇠*를 되찾는다)
*방법 A — VM/콘솔이 있을 때 (GRUB 편집):*
```
1. 부팅 중 GRUB 메뉴에서 e
2. linux 줄 끝에  init=/bin/bash   (Rocky/RHEL은  rd.break )
3. Ctrl-X 로 부팅
# (rd.break면 먼저 chroot /sysroot)
4. mount -o remount,rw /
# 새 비번
5. passwd root
# Rocky/RHEL SELinux 재라벨
6. touch /.autorelabel
7. exec /sbin/init  (또는 재부팅)
```
*방법 B — 콘솔이 막혔거나 부팅이 안 될 때 (라이브 USB chroot):*
```
1. Ventoy 메뉴 > GParted-q(또는 아무 라이브) 부팅
# 루트 파티션
2. sudo mount /dev/⟨sda2⟩ /mnt
3. for d in dev proc sys; do sudo mount --bind /$d /mnt/$d; done
4. sudo chroot /mnt passwd root
5. (Rocky면) sudo touch /mnt/.autorelabel
6. USB 떼고 부팅 → 새 비번으로 로그인
```

👀 **보이는 것:** 비밀번호를 **몰라도** 몇 줄이면 바꿀 수 있다. 디스크에 접근할 수 있으면 로그인 화면은 아무런 보호가 되지 못한다.

💡 **느낄 것:** 이것은 ①의 오프라인 디스크 편집을 **구조(救助)에 쓴 것**으로, `virt-customize --root-password`와 원리가 같다. 교훈도 분명하다. 콘솔이나 부팅 매체에 접근할 수 있으면 루트 권한을 가진 것과 같다. 그래서 BIOS/GRUB 암호, IPMI 보안, 디스크 암호화(LUKS), bastion(⑧, 관리 접속을 한곳으로 모으는 중계 서버)은 선택 사항이 아니라 **필수**다.

⚠️ 디스크 암호화(LUKS)가 걸려 있으면 방법 B도 패스프레이즈 없이는 열 수 없다. 이것이 방어의 요점이다.

---

🔀 **갈림길** — 다음엔 어디로?

- 되살린 서버의 문단속부터 → [③ ufw가 왜 뚫리지](lab-03.md)
- 루트 비밀번호 복구가 왜 곧 공격 기법인가 → [Part 1 §1.6](../01-image.md)
- Clone·V2P·Clonezilla를 언제 무엇으로 → [Part 1 §1.5](../01-image.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
