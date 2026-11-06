### ⑧ 준비 — SSH 키 만들기와 공개키 배포 {#lab-08-prep}

⌨️ **해보기**
```bash
# 1) 키 생성 — 요즘은 ed25519 (짧고·빠르고·안전). 레거시 호환 필요하면 rsa 4096
ssh-keygen -t ed25519 -C "⟨me@kreonet⟩"
#   (RSA가 필요하면)  ssh-keygen -t rsa -b 4096 -C "..."
# 지문: 256 SHA256:AbCd…  (ED25519)
ssh-keygen -lf ~/.ssh/id_ed25519.pub

# 2) 공개키를 '공개 URL'로 배포 — 한 번 올려두면 어디서나 당겨 쓴다
# GitHub가 내 공개키를 서빙
curl https://github.com/⟨myid⟩.keys
# Launchpad도
curl https://launchpad.net/~⟨myid⟩/+sshkeys

# 3) 서버(bastion)에서 그 키들을 authorized_keys로 당기기 (네트워크 필요)
# github.com/.keys 에서
ssh-import-id gh:⟨myid⟩
# launchpad 에서
ssh-import-id lp:⟨myid⟩
```

> 🔑 **공개키는 어디서 올리나** — 위 URL은 먼저 공개키를 계정에 등록해 두어야 내용이 나온다. `~/.ssh/id_ed25519.pub`의 내용(한 줄)을 붙여 넣는다.

> - GitHub: <https://github.com/settings/keys> → *New SSH key* ([GitHub 안내](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account))

> - Launchpad: <https://launchpad.net/~/+editsshkeys> ([Launchpad 안내](https://ubuntu.com/docs/launchpad/user/YourAccount/CreatingAnSSHKeyPair/))

>

> 개인키(`id_ed25519`, `.pub` 없는 쪽)는 절대 올리지 않는다.

👀 **보이는 것:** 공개키가 URL로 공개돼 있고, `ssh-import-id` 한 줄이면 서버에 등록됨. 이제 비밀번호 없이 **키로** 로그인.

💡 **키 타입 ≠ 지문 해시:** 키는 `ed25519`(종류), 지문은 `SHA256:…`(그 키를 요약한 해시)로, 서로 다른 층이다. "알고리즘이 sha256인가 ed25519인가" 하는 혼동은 이 구분으로 풀린다(옛 MD5 hex 지문은 이제 쓰지 않는다).

💡 **편하지만 신뢰 사슬이 생긴다:** 공개 URL에서 키를 가져온다는 것은 두 가지를 믿는다는 뜻이다. ① 그 **계정**: GitHub·Launchpad 계정이 탈취되면 공격자의 키가 추가될 수 있다. ② 그 공개 도메인(`github.com`)의 인증서를 발급한 **상위 CA**(인증기관), 정확히는 내 OS가 믿는 루트 CA 목록. 이 사슬이 깨지면 가짜 `github.com`이 가짜 키를 내줄 수 있다(⑪에서 직접 CA가 되어 본다). ①에서 이미지를 만들 때 `ssh-import-id`를 firstboot로 넣어 두면(ohmyzsh처럼 네트워크 필요) 장비마다 손대지 않아도 같은 접근 권한이 생긴다. 운영에는 편리하지만 그만큼 **감사 대상**이다.

## 실습⑧ 망 간 접점 통제 — egress 제한과 단일 진입점 {#lab-08}

🎯 **목표:** 격리망의 **egress를 기본 차단**하고 필요할 때만 토글, 그리고 **듀얼홈 bastion**(두 망에 모두 연결되어 관리 접속을 중계하는 서버)을 유일한 입구로 삼는다.

🖥️ **랩:** 방화벽 VM + 듀얼홈 bastion VM(Rocky, vmbr0+vmbr1) + 격리 LAN VM. (키·공개키는 위 "⑧ 준비"에서 완료)

⌨️ **(A) 방화벽에서 아웃바운드 기본 차단 + 임시 토글** (방화벽 웹 UI)
```
1. Firewall > Rules > LAN: 맨 아래 기본 Allow 규칙을 끄고 → "LAN net → any : block"
2. 꼭 필요한 것만 위에 허용:  LAN net → ⟨업데이트 서버/레지스트리⟩ : pass
3. "임시 허용" 규칙 하나를 Disabled로 만들어 두고 →
   필요할 때 체크 토글(enable) → 작업 끝나면 다시 끔
4. 격리 VM에서: 평소엔 외부 안 됨 → 토글 켠 순간만 나감
```

👀 **보이는 것:** 격리 VM `curl https://⟨아무사이트⟩`가 평소 **타임아웃** → 임시 규칙 켜면 **통과** → 끄면 다시 막힘. 허용 목록(업데이트 등)은 항상 됨.

⌨️ **(B) 듀얼홈 bastion — 외부에서 격리 VM으로 가는 유일한 문**

먼저 **bastion 쪽**에서 점프를 허용하는지 본다. `ProxyJump`는 bastion에 셸로 들어가지 않고, bastion의 sshd에게 지정한 주소로 TCP 연결을 이어 달라고 요청한다. 그래서 bastion sshd의 **TCP 포워딩이 허용**되어 있어야 한다.
```bash
# (bastion에서) 지금 실제로 적용되는 값 — jump 계정 기준으로
sudo sshd -T -C user=jump,host=x,addr=⟨내 PC IP⟩ | grep -Ei 'allowtcpforwarding|disableforwarding|permitopen'
# 점프 전용 계정에만 포워딩을 열고, 갈 수 있는 곳은 ssh(22)로 좁힌다
sudo tee /etc/ssh/sshd_config.d/10-jump.conf <<'EOF'
# 실습⑧: ProxyJump 허용 — 점프 전용 계정만
Match User jump
    AllowTcpForwarding yes
    # 점프로 이어 줄 수 있는 목적지: 어느 호스트든 22번만
    PermitOpen *:22
EOF
sudo sshd -t && sudo systemctl reload sshd
```

> 기본값이 `AllowTcpForwarding yes`이므로 갓 설치한 서버에서는 별도 설정 없이 된다. 문제는 **보안 강화 가이드에 따라 설정을 조인 서버**다. 점검 항목에 따라 `AllowTcpForwarding no`나 `DisableForwarding yes`가 들어가 있으면 점프가 이렇게 실패한다.
>
> ```text
> channel 0: open failed: administratively prohibited: open failed
> stdio forwarding failed
> ```
>
> 이 메시지가 나오면 내 PC가 아니라 **bastion의 sshd 설정**을 확인한다. 전체를 다시 열지 말고 위처럼 `Match`로 점프 계정에만 연다.

그다음 **내 쪽** 설정:
```bash
# bastion은 vmbr0(외부)·vmbr1(내부) 둘 다 꽂힌 멀티홈드 Rocky VM.
# ★ 멀티홈드 원칙: default route는 vmbr0 쪽 하나만, 내부 대역(192.0.2.0/24)은 vmbr1로 명시 라우트.
#   ip route add 192.0.2.0/24 dev ens19   (또는 nmcli c mod … +ipv4.routes)  ← 안 그러면 "어느 길로 나가지?" 모호
# 내 쪽: ~/.ssh/config 맨 위에 Include 한 줄 (이미 있으면 건너뛴다)
mkdir -p ~/.ssh/conf.d
if ! grep -qx 'Include conf.d/\*.conf' ~/.ssh/config 2>/dev/null; then
  { echo 'Include conf.d/*.conf'; echo; cat ~/.ssh/config 2>/dev/null; } > ~/.ssh/config.new
  mv ~/.ssh/config.new ~/.ssh/config; chmod 600 ~/.ssh/config
fi
# 이 실습의 접속 별명은 파일 하나로 — 팀이면 이 파일을 나눠 쓴다
cat > ~/.ssh/conf.d/osso-lab.conf <<'EOF'
Host bastion
  HostName ⟨bastion.wan⟩
  User jump
Host lanvm
  # vmbr1 내부 주소
  HostName ⟨192.0.2.5⟩
  ProxyJump bastion
EOF
# 실제로 적용되는 값 확인
ssh -G lanvm | grep -E '^(hostname|user|proxyjump) '
# bastion을 자동 경유해 격리 LAN VM 진입
ssh lanvm hostname
```

> 💡 `cat >> ~/.ssh/config`로 덧붙이지 않고 `conf.d/` 파일 하나로 나눈 이유: ①의 드롭인과 같다. 실습이 끝나면 파일 하나만 지우면 정리되고, 팀이라면 이 파일을 나눠 모두가 `ssh lanvm`이라는 **같은 이름**으로 들어간다. 점 파일(dotfile, 이름이 `.`으로 시작하는 개인 설정 파일)과 팀 공유는 [Part 1 §1.3](../01-image.md)에서 다룬다.

👀 **보이는 것:** `sshd -T`에 `allowtcpforwarding yes`·`permitopen *:22`. 격리 VM에 직접 `ssh`는 실패(경로 없음) → `ssh lanvm`은 bastion을 거쳐 성공. (해 보고 싶다면: `10-jump.conf`에서 `yes`를 `no`로 바꾸고 reload → 같은 명령이 `administratively prohibited`로 실패.)

💡 **느낄 것:** 격리망을 쓰려면 **나가는 길**과 **들어오는 문**을 각각 통제한다. egress는 평소 잠그고 필요할 때만 토글(폐쇄망 조이기), 진입은 bastion **한 곳**으로 좁힌다(망 간 접점). 격리망을 두면서도 안전하게 쓰는 방법은 결국 이 두 가지다. ⑥(호스트 하나)에서 ⑧(망 전체 경계)로 생각의 범위가 넓어진다.

---

🔀 **갈림길** — 다음엔 어디로?

- egress를 AI 고도에서 → [⑭ AI 배포 캡스톤](lab-14.md)
- 서비스를 실행해 보는 2일차로 → [⑨ compose](lab-09.md)
- ssh 키·bastion·접근 통제 → [Part 6](../06-access.md)
- 듀얼홈·격리망의 원리 → [Part 3](../03-network.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
