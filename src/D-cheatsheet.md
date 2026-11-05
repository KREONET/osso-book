# 부록 D. 명령어 치트시트 (Runbook)

실습에서 쓴 핵심 커맨드 모음입니다. `⟨…⟩` 는 본인 값으로 바꾸세요.

## 1. 이미지 (libguestfs / qemu)

```bash
virt-inspector -a ⟨image.img⟩ | head -40     # OS·설치 앱 목록 (부팅 없이)
virt-ls  -a ⟨image.img⟩ /etc | head          # 디렉터리 보기
virt-cat -a ⟨image.img⟩ /etc/os-release      # 파일 내용 뽑기
```

```bash
sudo modprobe nbd max_part=8                 # nbd 모듈 로드
sudo qemu-nbd -c /dev/nbd0 ⟨image.qcow2⟩     # 이미지를 블록장치로 연결
sudo mount /dev/nbd0p1 /mnt                  # 마운트해서 파일 확인·수정
sudo umount /mnt && sudo qemu-nbd -d /dev/nbd0   # 반드시 해제
```

```bash
virt-customize -a ⟨image.img⟩ \
  --root-password file:⟨pw.txt⟩ \
  --install ⟨qemu-guest-agent⟩ \
  --firstboot-command '⟨systemctl enable --now ssh⟩'   # 이미지 사전 커스터마이즈
virt-sysprep -a ⟨image.img⟩                  # 정체성 리셋(machine-id·hostkey·로그)
qemu-img convert -O qcow2 ⟨in.img⟩ ⟨out.qcow2⟩   # 포맷 변환 (raw/vmdk/qcow2)
qemu-img info ⟨image.qcow2⟩                  # 포맷·크기 확인
```

## 2. 디스크 / 부팅

```bash
sudo sh Ventoy2Disk.sh -i /dev/⟨sdX⟩         # Ventoy 설치 (USB 내용 삭제 주의)
# 이후 ISO 파일을 USB에 복사만 하면 부팅 메뉴에 나타남
```

```text
Clonezilla: savedisk    ⟨이미지명⟩ ⟨sda⟩     # 디스크 → 이미지 백업
Clonezilla: restoredisk ⟨이미지명⟩ ⟨sda⟩     # 이미지 → 디스크 복원
```

루트 비밀번호 분실 복구: GRUB 메뉴에서 `e` 로 편집, `linux` 줄 끝에 추가 후 `Ctrl+X`.

```text
init=/bin/bash
```

```bash
mount -o remount,rw /       # 루트를 쓰기 가능으로
passwd ⟨user⟩               # 비밀번호 재설정
exec /sbin/init             # 정상 부팅 이어가기
```

## 3. Docker / Compose

```bash
docker compose up -d                  # 백그라운드 기동
docker compose ps                     # 상태
docker compose logs -f ⟨svc⟩          # 로그 따라가기
docker compose exec ⟨svc⟩ sh          # 컨테이너 안으로
docker compose build                  # 이미지 빌드
```

```bash
alias dc='docker compose'             # 자주 쓰는 alias 패턴
alias dcl='docker compose logs -f'
docker run --rm ⟨image⟩ cat /etc/⟨app⟩/⟨default.conf⟩ > ⟨default.conf⟩   # 기본 설정 추출
```

```yaml
# compose.yml: 밖으로 못 나가는 격리 네트워크
networks:
  backend:
    internal: true
```

## 4. SSH / OpenSSL

```bash
ssh-keygen -t ed25519 -C "⟨me@example⟩"      # 키 생성
ssh-keygen -lf ~/.ssh/id_ed25519.pub         # 지문 확인
ssh-import-id gh:⟨github-id⟩                 # GitHub 공개키 가져오기 (lp: 는 Launchpad)
ssh -J ⟨bastion⟩ ⟨target⟩                    # ProxyJump 경유 접속
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

## 5. 방화벽

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing               # egress 기본 차단
sudo ufw allow out 53                        # 필요한 나가는 길만 허용
sudo ufw allow from ⟨10.0.0.0/24⟩ to any port 22
sudo ufw enable && sudo ufw status verbose
```

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=⟨public⟩ --add-service=https --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all-policies        # policy: zone 간 트래픽 규칙
```

```bash
sudo nft list ruleset                        # 실제 적용된 규칙 전체
sudo iptables-nft -L DOCKER-USER -n -v       # Docker 트래픽이 지나는 사용자 체인
```

## 6. 네트워크

```bash
ip route                                     # 라우팅 테이블
ip -br addr                                  # 주소 요약
nmcli con show                               # NetworkManager 연결 목록
nmcli con mod ⟨con⟩ ipv4.addresses ⟨10.0.0.5/24⟩ ipv4.method manual
sudo wg show                                 # WireGuard 상태
ping ⟨10.10.1.5⟩                             # VPN 후 격리 LAN 도달 확인
ping -M do -s 8972 ⟨target⟩                  # MTU 9000 확인(DF 비트, 8972+28)
```

관련: [부록 F 용어집](F-glossary.md) · [Part 2](02-firewall.md)
