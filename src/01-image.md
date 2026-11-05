# Part 1. 이미지 — 서버를 "만들어 두고" 쓴다

> 담당 랩 ①②. 랩에서 손으로 만든 뒤, 여기서 **왜 그렇게 하는지**를 본다.

## 1.1 왜 ISO를 안 까는가 — 골든 이미지와 선언형 운영
> 📝 작성 예정.

## 1.2 클라우드 이미지 해부 — qcow2/raw, OVA/OVF, thin vs thick
> 📝 작성 예정. (사내 위키 *인턴 읽을거리* 참조)

## 1.3 끄고 고치기 — libguestfs와 오프라인이 안전한 이유
`virt-inspector`/`virt-ls`/`virt-cat`, `qemu-nbd` 마운트.
> 📝 작성 예정.

## 1.4 정체성 리셋 — virt-sysprep, machine-id, SSH hostkey, SELinux .autorelabel
> 📝 작성 예정.

## 1.5 배포의 세 길과 복구 — PVE Clone / V2P / Clonezilla
> 📝 작성 예정.

## 1.6 열쇠를 잃었을 때 — 루트 비번 복구와 그 함의
콘솔=루트, BIOS/GRUB·IPMI·LUKS. (랩②ⓕ)
> 📝 작성 예정.

## 1.7 하드웨어 신원 — inxi·IPMI·서비스태그·자산 스티커
사내 납품 가이드의 인벤토리 절차 → ohmyzsh 자산 태그.
> 📝 작성 예정.

## 1.8 재현가능 환경 — 패키지 관리자 지형과 marimo
apt → pip/uv → conda(anaconda▶miniforge) → 컨테이너.
> 📝 작성 예정.

## 1.9 Ventoy 한 자루 — 부팅도구+이미지 한 USB
`make-usb-ventoy.sh` 해설.
> 📝 작성 예정.
