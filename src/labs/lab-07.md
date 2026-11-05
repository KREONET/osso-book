## 실습⑦ 격리망 게이트웨이 — 방화벽 세우고 VPN으로 내부에 들어가기

🎯 **목표:** pfSense/OPNsense를 **WAN↔격리 LAN** 사이 관문으로 세우고, VPN으로 내부(섬)에 접속.

🖥️ **랩:** 방화벽 VM(WAN=vmbr0, LAN=vmbr1) + 격리 LAN VM(vmbr1) + lubuntu desktop VM(클라이언트). **전부 PVE noVNC.**

> **망 그림:** `인터넷 ─ vmbr0(WAN) ─ [방화벽] ─ vmbr1(LAN, 물리 NIC 없음) ─ 격리 VM`

> 격리 VM은 vmbr1에만 있어 **밖에서 직접 못 닿는다.** VPN으로 방화벽을 통과해야 보임.

⌨️ **(1) PVE에서 방화벽 VM 다리 놓기** — VM 하드웨어에 NIC 2장
```
net0 → vmbr0  (WAN, provider 네트워크)
net1 → vmbr1  (LAN, 어느 물리 NIC에도 안 붙은 격리 브리지)
```

⌨️ **(2) 방화벽 웹 UI에서 VPN 올리기** (OpenVPN · WireGuard 둘 다 체험)
```
OpenVPN:  VPN > OpenVPN > 서버 생성(Wizard) → 클라이언트 프로파일(.ovpn) 내보내기
WireGuard: System>Firmware>Plugins>os-wireguard 설치 →
           Instances 서버 생성(Listen Port ⟨커스텀⟩) → Peer 추가 → 설정 내보내기
방화벽 규칙: WAN에서 VPN 포트 허용 / VPN→LAN 통과 허용
```

⌨️ **(3) lubuntu에서 프로파일 받아 접속** (PVE noVNC 안)
```bash
# 방화벽 WAN 주소로 웹 접속해 프로파일 다운로드 후
sudo openvpn --config ⟨client.ovpn⟩          # 또는 WireGuard: wg-quick up ⟨wg0⟩
ping ⟨10.10.1.5⟩          # 격리 LAN VM — VPN 전엔 안 되던 게 이제 됨
ssh ⟨user@10.10.1.5⟩      # 내부로 진입
```

👀 **보이는 것:** VPN **전**엔 격리 VM에 `ping`/`ssh` 안 됨 → VPN **후** 바로 닿음. 방화벽 대시보드에 터널 핸드셰이크 표시.

💡 **느낄 것:** 내부망을 **섬**으로 만들고(물리 NIC 없는 vmbr1), 들어가는 **유일한 다리**가 VPN이다. "원격근무로 사내망 들어간다"가 이 구조. 폰 없이 lubuntu로 전 과정이 PVE 안에서 된다.

⚠️ 폰으로도 가능(WireGuard 앱+QR) — 단 교육장에서 폰→방화벽 외부경로가 될 때만. 기본은 lubuntu.

