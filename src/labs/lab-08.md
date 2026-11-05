### ⑧ 준비 — 내 열쇠 만들고 공개키 뿌리기 (bastion에 들어가려면 먼저 이게 있어야)

⌨️ **해보기**
```bash
# 1) 키 생성 — 요즘은 ed25519 (짧고·빠르고·안전). 레거시 호환 필요하면 rsa 4096
ssh-keygen -t ed25519 -C "⟨me@kreonet⟩"
#   (RSA가 필요하면)  ssh-keygen -t rsa -b 4096 -C "..."
ssh-keygen -lf ~/.ssh/id_ed25519.pub        # 지문: 256 SHA256:AbCd…  (ED25519)

# 2) 공개키를 '공개 URL'로 배포 — 한 번 올려두면 어디서나 당겨 쓴다
curl https://github.com/⟨msjang⟩.keys          # GitHub가 내 공개키를 서빙
curl https://launchpad.net/~⟨msjang⟩/+sshkeys  # Launchpad도

# 3) 서버(bastion)에서 그 키들을 authorized_keys로 당기기 (네트워크 필요)
ssh-import-id gh:⟨msjang⟩                      # github.com/.keys 에서
ssh-import-id lp:⟨msjang⟩                      # launchpad 에서
```

👀 **보이는 것:** 공개키가 URL로 공개돼 있고, `ssh-import-id` 한 줄이면 서버에 등록됨. 이제 비번 없이 **키로** 로그인.

💡 **키 타입 ≠ 지문 해시:** 키는 `ed25519`(종류), 지문은 `SHA256:…`(그 키를 요약한 해시) — 서로 다른 층. "알고리즘이 sha256/ed…"는 여기서 갈린다(옛 MD5 hex 지문은 이제 안 씀).

💡 **편하지만 신뢰 사슬이 생긴다:** 공개 URL에서 키를 당긴다 = 그 **계정 + TLS(CA)**를 믿는 것(⑪과 연결). GitHub 계정 털리면 키가 추가될 수 있다. 그래서 **① 이미지 만들 때** `ssh-import-id`를 **firstboot**로 심어두면(ohmyzsh처럼 네트워크 필요) 장비마다 손 안 대고 동일한 접근권 — 운영 편의이자 **감사 대상**.

## 실습⑧ 망간접점 통제 — 나가는 길 조이기 + 들어오는 문 하나로

🎯 **목표:** 격리망의 **egress를 기본 차단**하고 필요할 때만 토글, 그리고 **듀얼홈 bastion**을 유일한 입구로.

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
```bash
# bastion은 vmbr0(외부)·vmbr1(내부) 둘 다 꽂힌 멀티홈드 Rocky VM.
# ★ 멀티홈드 원칙: default route는 vmbr0 쪽 하나만, 내부 대역(10.10.1.0/24)은 vmbr1로 명시 라우트.
#   ip route add 10.10.1.0/24 dev ens19   (또는 nmcli c mod … +ipv4.routes)  ← 안 그러면 "어느 길로 나가지?" 모호
# 내 쪽 ~/.ssh/config:
cat >> ~/.ssh/config <<'EOF'
Host bastion
  HostName ⟨bastion.wan⟩
  User jump
Host lanvm
  HostName ⟨10.10.1.5⟩        # vmbr1 내부 주소
  ProxyJump bastion
EOF
ssh lanvm hostname             # bastion을 자동 경유해 격리 LAN VM 진입
```

👀 **보이는 것:** 격리 VM에 직접 `ssh`는 실패(경로 없음) → `ssh lanvm`은 bastion을 거쳐 성공.

💡 **느낄 것:** 격리망을 쓰려면 **나가는 길**과 **들어오는 문**을 각각 통제한다. egress는 평소 잠그고 필요할 때만 토글(폐쇄망 조이기), 진입은 bastion **한 곳**으로 좁힌다(망간접점). 이 둘이 "격리망을 두되 안전하게 쓰는" 실체. ⑥(호스트 하나)에서 ⑧(망 전체 경계)로 사고가 올라간다.

---

