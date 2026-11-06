# 부록 D. 명령어 치트시트 (Runbook) {#appendix-d}

실습에서 쓴 주요 명령어 모음입니다. `⟨…⟩` 는 본인 값으로 바꾸세요.

## 1. 이미지 (libguestfs / qemu) {#d-1-image}

```bash
# OS·설치 앱 목록 (부팅 없이)
virt-inspector -a ⟨image.img⟩ | head -40
# 디렉터리 보기
virt-ls  -a ⟨image.img⟩ /etc | head
# 파일 내용 뽑기
virt-cat -a ⟨image.img⟩ /etc/os-release
```

```bash
# nbd 모듈 로드
sudo modprobe nbd max_part=8
# 이미지를 블록장치로 연결
sudo qemu-nbd -c /dev/nbd0 ⟨image.qcow2⟩
# 마운트해서 파일 확인·수정
sudo mount /dev/nbd0p1 /mnt
# 반드시 해제
sudo umount /mnt && sudo qemu-nbd -d /dev/nbd0
```

```bash
# 이미지 사전 커스터마이즈
virt-customize -a ⟨image.img⟩ \
  --root-password file:⟨pw.txt⟩ \
  --install ⟨qemu-guest-agent⟩ \
  --firstboot-command '⟨systemctl enable --now ssh⟩'
# 정체성 리셋(machine-id·hostkey·로그)
virt-sysprep -a ⟨image.img⟩
# 포맷 변환 (raw/vmdk/qcow2)
qemu-img convert -O qcow2 ⟨in.img⟩ ⟨out.qcow2⟩
# 포맷·크기 확인
qemu-img info ⟨image.qcow2⟩
```

## 2. 디스크 / 부팅 {#d-2-disk}

```bash
# Ventoy 설치 (USB 내용 삭제 주의)
sudo sh Ventoy2Disk.sh -i /dev/⟨sdX⟩
# 이후 ISO 파일을 USB에 복사만 하면 부팅 메뉴에 나타남
```

```text
# 디스크 → 이미지 백업
Clonezilla: savedisk    ⟨이미지명⟩ ⟨sda⟩
# 이미지 → 디스크 복원
Clonezilla: restoredisk ⟨이미지명⟩ ⟨sda⟩
```

루트 비밀번호 분실 복구: GRUB 메뉴에서 `e` 로 편집, `linux` 줄 끝에 추가 후 `Ctrl+X`.

```text
init=/bin/bash
```

```bash
# 루트를 쓰기 가능으로
mount -o remount,rw /
# 비밀번호 재설정
passwd ⟨user⟩
# 정상 부팅 이어가기
exec /sbin/init
```

## 3. Docker / Compose {#d-3-docker}

```bash
# 백그라운드 기동
docker compose up -d
# 상태
docker compose ps
# 로그 따라가기
docker compose logs -f ⟨svc⟩
# 컨테이너 안으로
docker compose exec ⟨svc⟩ sh
# 이미지 빌드
docker compose build
```

```bash
# 자주 쓰는 alias 패턴
alias dc='docker compose'
alias dcl='docker compose logs -f'
# 기본 설정 추출
docker run --rm ⟨image⟩ cat /etc/⟨app⟩/⟨default.conf⟩ > ⟨default.conf⟩
```

```yaml
# compose.yml: 밖으로 못 나가는 격리 네트워크
networks:
  backend:
    internal: true
```

## 4. SSH / OpenSSL {#d-4-ssh}

```bash
# 키 생성
ssh-keygen -t ed25519 -C "⟨me@example⟩"
# 지문 확인
ssh-keygen -lf ~/.ssh/id_ed25519.pub
# GitHub 공개키 가져오기 (lp: 는 Launchpad)
ssh-import-id gh:⟨github-id⟩
# ProxyJump 경유 접속
ssh -J ⟨bastion⟩ ⟨target⟩
```

```bash
# CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -days 3650 -out ca.crt -subj "/CN=⟨lab-ca⟩"
# 서버
openssl genrsa -out srv.key 2048
openssl req -new -key srv.key -subj "/CN=⟨srv.lab.local⟩" -out srv.csr
openssl x509 -req -in srv.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -out srv.crt
# 클라이언트 (mTLS)
openssl genrsa -out cli.key 2048
openssl req -new -key cli.key -subj "/CN=⟨alice⟩" -out cli.csr
openssl x509 -req -in cli.csr -CA ca.crt -CAkey ca.key -days 365 -out cli.crt
```

## 5. 방화벽 {#d-5-firewall}

```bash
sudo ufw default deny incoming
# egress 기본 차단
sudo ufw default deny outgoing
# 필요한 나가는 길만 허용
sudo ufw allow out 53
sudo ufw allow from ⟨192.0.2.0/24⟩ to any port 22
sudo ufw enable && sudo ufw status verbose
```

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=⟨public⟩ --add-service=https --permanent
sudo firewall-cmd --reload
# policy: zone 간 트래픽 규칙
sudo firewall-cmd --list-all-policies
```

```bash
# 실제 적용된 규칙 전체
sudo nft list ruleset
# Docker 트래픽이 지나는 사용자 체인
sudo iptables-nft -L DOCKER-USER -n -v
```

## 6. 네트워크 {#d-6-network}

```bash
# 라우팅 테이블
ip route
# 주소 요약
ip -br addr
# NetworkManager 연결 목록
nmcli con show
nmcli con mod ⟨con⟩ ipv4.addresses ⟨192.0.2.5/24⟩ ipv4.method manual
# WireGuard 상태
sudo wg show
# VPN 후 격리 LAN 도달 확인
ping ⟨192.0.2.5⟩
# MTU 9000 확인(DF 비트, 8972+28)
ping -M do -s 8972 ⟨target⟩
```

## 7. 서비스 (systemd) {#d-7-systemd}

옛 문서에는 `service`·`/etc/init.d/` 명령이 많습니다. 현재 배포판은 systemd를 쓰며, 옛 명령은 대개 systemd로 넘겨 주는 호환용입니다.

| | 옛 방식 (init.d) | 지금 (systemd) |
| --- | --- | --- |
| 재시작 | `service sshd restart` | `systemctl restart sshd` |
| 상태 | `service sshd status` | `systemctl status sshd` |
| 부팅 시 자동 시작 | `chkconfig sshd on` | `systemctl enable sshd` |
| 로그 | `/var/log/…` 파일 | `journalctl -u sshd` |

```bash
# 설정 문법 검사 후 재시작 (sshd 예)
sudo sshd -t
sudo systemctl restart sshd
# 이 서비스의 최근 로그
journalctl -u sshd -n 50
# 부팅 때 켜지는가
systemctl is-enabled sshd
```

데비안·우분투에서는 서비스 이름이 `ssh`입니다(`systemctl restart ssh`). 프로세스 트리 관점에서 systemd가 무엇을 바꿨는지는 [Part 4](04-container.md)에 있습니다.

관련: [부록 F 용어집](F-glossary.md) · [Part 2](02-firewall.md)
