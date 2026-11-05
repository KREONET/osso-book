## 실습① 클라우드 이미지 — 받아서, 안 켜고 열어보고, 고친다
🎯 **목표:** 부팅 없이 이미지 내부를 **탐색**하고 계정·정체성까지 **패치**한다.
🖥️ **랩:** 작업 VM(libguestfs·qemu-utils 설치됨). 이미지는 파일 하나로 다룬다.

⌨️ **받기**
```bash
wget ⟨https://⟨mirror.example⟩/.../jammy-server-cloudimg-amd64.img⟩ -O ⟨image.img⟩
```
⌨️ **탐색 — 안 켜고 안을 들여다본다(읽기)**
```bash
virt-inspector -a ⟨image.img⟩ | head -40     # OS·설치앱 목록
virt-ls  -a ⟨image.img⟩ /etc | head          # 디렉터리 보기 (부팅 없이)
virt-cat -a ⟨image.img⟩ /etc/os-release       # 파일 내용 뽑기
# 진짜 파일시스템으로 마운트해서 보기 (qemu-nbd)
sudo modprobe nbd max_part=8
sudo qemu-nbd --connect=/dev/nbd0 ⟨image.img⟩
sudo mount /dev/nbd0p1 /mnt && ls /mnt/etc    # 디스크를 폴더처럼
sudo umount /mnt; sudo qemu-nbd --disconnect /dev/nbd0
```
⌨️ **패치 — 계정·패키지·운영환경·정체성을 심는다(쓰기)**
```bash
virt-customize -a ⟨image.img⟩ \
  --root-password password:⟨labpass⟩ \
  --run-command 'useradd -m -s /bin/bash ⟨labuser⟩' \
  --password ⟨labuser⟩:password:⟨labpass⟩ \
  --install htop,ncdu \
  --run-command 'systemctl disable snapd.service' \
  --write /etc/tag1:"⟨Dell R740⟩" \          # 자산 태그: 하드웨어 모델
  --write /etc/tag2:"⟨ASSET-00123⟩" \        # 자산 태그: 자산 ID
  --firstboot-command \
    'curl -s https://raw.githubusercontent.com/KREONET/ohmyzsh/main/install.sh | sh -s -'
virt-sysprep -a ⟨image.img⟩                   # 정체성 리셋(machine-id·hostkey·로그)
# ⚠️ Rocky/RHEL 계열이면 다음 부팅 때 SELinux 재라벨:
# virt-customize -a ⟨rocky.qcow2⟩ --touch /.autorelabel
```
> 오프라인 편집 중엔 네트워크가 없다 → ohmyzsh는 **`--firstboot-command`**로 **첫 부팅 때** 설치되게 심는다(egress 허용 필요 — Day1 ⑥과 연결).
👀 **보이는 것:** inspector가 OS·앱을 뱉고, `virt-ls`/`qemu-nbd`로 안의 파일이 그대로 보임. customize가 `Installing packages…`·`Writing /etc/tag1…`, sysprep이 `Performing "machine-id"…` 수십 줄. **부팅 후** zsh 프롬프트에 `[TAG1 Dell R740 | TAG2 ASSET-00123 | HNUM 42]` 식으로 자산 태그가 뜬다.
💡 **느낄 것:** 이미지가 **켜야만 보는 블랙박스가 아니다.** 안 켜고 열어보고(virt-ls·qemu-nbd) 고친다(virt-customize) — 계정·패키지뿐 아니라 **운영 편의 환경까지** 미리 심는다. 그 프롬프트가 "**지금 내가 어느 장비에 있나**"를 대신 말해준다(①②의 머신 식별과 한 쌍). `!N`으로 과거 명령 재실행은 덤.
⚠️ 모든 오프라인 편집은 이미지가 **꺼진 상태**에서만.

