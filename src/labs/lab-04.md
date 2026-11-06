## 실습④ firewalld trusted zone — 영역 단위 관리 {#lab-04}

🎯 **목표:** 인터페이스/소스를 영역(zone)에 넣어 규칙을 영역 단위로 관리한다.

🖥️ **랩:** 작업 VM(또는 Rocky 계열 VM).

⌨️ **해보기**
```bash
sudo firewall-cmd --get-active-zones
# 내부망을 통째로 신뢰 영역에
sudo firewall-cmd --permanent --zone=trusted --add-source=⟨192.0.2.0/24⟩
# 공개 영역엔 꼭 필요한 서비스만
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --zone=trusted --list-all
```

👀 **보이는 것:** trusted에 소스 대역이, public에 https만. 내부망에서는 모두 접속되고 외부에서는 https만 된다.

💡 **느낄 것:** 규칙을 포트 하나하나가 아니라 **"이 대역은 믿는다/안 믿는다"** 로 생각하게 된다. ufw(데비안계)와 firewalld(레드햇계)는 사고방식이 다르다.
  단, Docker publish 포트에는 zone 규칙이 적용되지 않는다. 그 트래픽은 FORWARD로 가므로 **egress=docker인 policy**로 막고, 기준도 호스트 포트가 아닌 **컨테이너 포트**로 잡는다(③의 ufw 함정과 원인이 같다).

---

🔀 **갈림길** — 다음엔 어디로?

- 들어오는 쪽을 평판으로 미리 거르기 → [⑤ 인바운드 선차단](lab-05.md)
- 이번엔 나가는 쪽을 막기 → [⑥ egress 화이트리스팅](lab-06.md)
- zone과 policy, 도커와 firewalld → [Part 2](../02-firewall.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
