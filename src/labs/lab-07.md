## 실습⑦ 격리망 게이트웨이 — 방화벽 구성과 VPN 내부 접속 {#lab-07}

🎯 **목표:** pfSense/OPNsense를 **WAN↔격리 LAN** 사이 관문으로 세우고, VPN으로 내부(섬)에 접속.

🖥️ **랩:** 방화벽 VM(WAN=vmbr0, LAN=vmbr1) + 격리 LAN VM(vmbr1) + lubuntu desktop VM(클라이언트). **모두 PVE noVNC에서 진행한다.**

> **망 그림:** `인터넷 ─ vmbr0(WAN) ─ [방화벽] ─ vmbr1(LAN, 물리 NIC 없음) ─ 격리 VM`

> 격리 VM은 vmbr1에만 있어 **밖에서 직접 접근할 수 없다.** VPN으로 방화벽을 통과해야 보인다.

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
# 또는 WireGuard: wg-quick up ⟨wg0⟩
sudo openvpn --config ⟨client.ovpn⟩
# 격리 LAN VM — VPN 전엔 안 되던 게 이제 됨
ping ⟨192.0.2.5⟩
# 내부로 진입
ssh ⟨user@192.0.2.5⟩
```

👀 **보이는 것:** VPN **연결 전**에는 격리 VM에 `ping`/`ssh`가 안 됨 → VPN **연결 후** 바로 접속됨. 방화벽 대시보드에 터널 핸드셰이크 표시.

💡 **느낄 것:** 내부망을 **섬**으로 만들고(물리 NIC 없는 vmbr1), 들어가는 **유일한 다리**가 VPN이다. 원격 근무자가 사내망에 접속하는 방식이 이 구조다. 휴대전화 없이 lubuntu만으로 전 과정을 PVE 안에서 진행할 수 있다.

⚠️ 휴대전화로도 할 수 있다(WireGuard 앱+QR). 단, 교육장에서 휴대전화→방화벽 외부 경로가 열려 있을 때만 된다. 기본은 lubuntu.

---

🔀 **갈림길** — 다음엔 어디로?

- 격리망에 키를 배포하고 접속 경로를 하나로 → [⑧ 망 간 접점·bastion](lab-08.md)
- VPN·격리망 설계의 원리 → [Part 3](../03-network.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
