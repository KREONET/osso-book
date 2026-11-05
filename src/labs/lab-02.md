## 실습② 배포·복원 — Ventoy · 클론 · V2P · Clonezilla · 루트복구 (디스크를 날려봐도, 열쇠를 잃어도 되살린다)

🎯 **목표:** 준비한 이미지를 디스크에 붓고(V2P), 지금 상태를 백업·복원(Clonezilla)하고, 비번까지 잃어도 되찾는다 — **되돌리기 네 갈래**.

🖥️ **랩:** 작업 VM + `usb-ventoy.qcow2` + (삭제·추가할) 디스크. 전부 PVE.

⌨️ **ⓐ Ventoy USB 하나에 부팅도구 + 이미지 싣기** (작업 VM)
```bash
sudo ventoy -i /dev/⟨sdb⟩          # usb-ventoy.qcow2 에 Ventoy 설치
#   필요시 -r ⟨4096⟩ 로 뒤에 예비(데이터) 파티션 공간 확보
sudo mount /dev/⟨sdb1⟩ /mnt
sudo cp gparted-q.iso clonezilla.iso ⟨image.img⟩ /mnt/ ; sync   # 부팅 ISO들 + 패치 이미지
```

👀 USB 데이터 파티션에 `gparted-q.iso`와 ①의 패치 이미지가 **같이** 들어감.

💡 **USB 하나가 부팅도구와 페이로드를 둘 다 싣는다.** ISO는 복사만 해도 Ventoy 메뉴에 뜬다.

⌨️ **ⓑ 안전망 — 이 머신 클론(백업)** (PVE 웹)
```
작업 VM 우클릭 > Clone > Full Clone  →  되돌릴 수 있는 백업 확보
```

💡 뭘 날리기 전에 **클론 = undo 버튼**부터 만든다.

⌨️ **ⓒ 재난 — 메인 디스크 날리고 부팅** (PVE 웹)
```
VM > Hardware > 메인 디스크 선택 > Detach → Remove
→ 부팅  →  "No bootable device" (OS가 없으니 당연)
```

👀 멀쩡하던 서버가 안 뜬다. 디스크가 OS다.

⌨️ **ⓓ 복원 — 빈 디스크 추가 + USB로 부팅해 V2P**
```bash
# PVE: Hardware > Add > Hard Disk(20G), 그리고 usb-ventoy 디스크 붙임 → USB로 부팅
#      Ventoy 메뉴 > GParted-q 선택 → 라이브 터미널:
lsblk                               # 타겟 빈 디스크(⟨sda⟩)·Ventoy(⟨sdb⟩) 확인
sudo mount /dev/⟨sdb1⟩ /mnt
sudo qemu-img convert -p -f qcow2 -O raw /mnt/⟨image.img⟩ /dev/⟨sda⟩
# ISO·USB 떼고 → 새 디스크로 부팅
```
그리고 부팅 후 — **되살아났는지 확인**
```bash
htop --version            # ①에서 넣은 패치가 그대로 있다
cat /etc/machine-id       # sysprep 덕에 새 값(클론과 안 겹침)
sudo hostnamectl set-hostname ⟨042-u24-web⟩   # 새 장비엔 새 이름 — ⟨IP끝 3자리⟩-⟨OS⟩-⟨용도⟩
```

> 💡 VM이 많아지면 `ubuntu@ubuntu`로는 헷갈린다. **`⟨IP 끝자리 3자리⟩-⟨u24 같은 OS약어⟩-⟨용도⟩`** 처럼 이름에 정보를 넣으면 프롬프트만 보고 "어디냐·apt냐 dnf냐·재부팅해도 되냐"가 보인다. OS 버전·IP는 어차피 `/etc/os-release`·`ip a`로 누구나 보는 정보라 보안 손해는 거의 없고 편의는 크다([Part 1 §1.7](../01-image.md)).

👀 **보이는 것:** 처음엔 부팅 실패 → convert 100% → 디스크·USB 떼고 부팅하면 **OS가 뜨고**, 내 패치(htop)와 **새 정체성**이 확인됨. 설치 마법사 한 번 안 거쳤다.

💡 **느낄 것:** 클론은 **되돌리기**, V2P는 **맨 디스크에 붓기**. 준비된 이미지만 있으면 디스크를 날려도 분 단위로 되살린다 — 현장에선 이 Ventoy USB를 들고 가 꽂으면 끝. ①의 오프라인 편집이 여기까지 한 줄기로 이어진다.

⚠️ `qemu-img convert` **대상 `/dev/`를 반드시 확인.** 엉뚱한 디스크를 고르면 통째로 덮인다.

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

💡 **느낄 것:** PVE Clone은 **하이퍼바이저가 있어야** 눌리는 버튼. 물리 서버엔 그 버튼이 없다 — **Clonezilla가 어디서나(라이브 USB) 그 버튼이 된다.** V2P(골든 이미지 붓기)와 달리 이건 **지금 이 서버의 상태**를 통째로 뜬다.

⚠️ Clonezilla 저장소 FS는 ext4/ntfs/fat 권장(exFAT 호환 이슈). 데이터 파티션을 저장소로 쓰려면 `DATA_FS=ext4`로 만든 USB 사용.

⌨️ **ⓕ 열쇠를 잃었을 때 — 루트 비번 복구** (같은 USB로, 이번엔 시스템이 아니라 *열쇠*를 되찾는다)
*방법 A — VM/콘솔이 있을 때 (GRUB 편집):*
```
1. 부팅 중 GRUB 메뉴에서 e
2. linux 줄 끝에  init=/bin/bash   (Rocky/RHEL은  rd.break )
3. Ctrl-X 로 부팅
4. mount -o remount,rw /          # (rd.break면 먼저 chroot /sysroot)
5. passwd root                    # 새 비번
6. touch /.autorelabel            # Rocky/RHEL SELinux 재라벨
7. exec /sbin/init  (또는 재부팅)
```
*방법 B — 콘솔이 막혔거나 부팅이 안 될 때 (라이브 USB chroot):*
```
1. Ventoy 메뉴 > GParted-q(또는 아무 라이브) 부팅
2. sudo mount /dev/⟨sda2⟩ /mnt   # 루트 파티션
3. for d in dev proc sys; do sudo mount --bind /$d /mnt/$d; done
4. sudo chroot /mnt passwd root
5. (Rocky면) sudo touch /mnt/.autorelabel
6. USB 떼고 부팅 → 새 비번으로 로그인
```

👀 **보이는 것:** 비번을 **몰라도** 몇 줄이면 바뀐다. 디스크를 손에 쥔 순간 로그인 화면은 장식이 된다.

💡 **느낄 것:** 이건 ①의 **오프라인 디스크 편집을 구조(救助)로 돌린 것** — `virt-customize --root-password`와 뿌리가 같다. 그리고 **섬뜩한 교훈**: 콘솔/부팅매체 접근 = 루트. 그래서 **BIOS/GRUB 암호·IPMI 보안·디스크 암호화(LUKS)·bastion(⑧)**이 "있으면 좋은"이 아니라 **필수**다.

⚠️ 디스크 암호화(LUKS)가 걸려 있으면 방법 B도 패스프레이즈 없이는 못 연다 — 그게 방어의 핵심.

---

🔀 **갈림길** — 다음엔 어디로?

- 되살린 서버의 문단속부터 → [③ ufw가 왜 뚫리지](lab-03.md)
- 루트 비번 복구가 왜 곧 공격 기법인가 → [Part 1 §1.6](../01-image.md)
- Clone·V2P·Clonezilla를 언제 무엇으로 → [Part 1 §1.5](../01-image.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
