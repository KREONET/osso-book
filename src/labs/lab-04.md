## 실습④ firewalld trusted zone — 영역으로 관리한다

🎯 **목표:** 인터페이스/소스를 영역에 넣어 규칙을 영역 단위로.

🖥️ **랩:** 작업 VM(또는 Rocky 계열 VM).

⌨️ **해보기**
```bash
sudo firewall-cmd --get-active-zones
# 내부망을 통째로 신뢰 영역에
sudo firewall-cmd --permanent --zone=trusted --add-source=⟨10.10.0.0/24⟩
# 공개 영역엔 꼭 필요한 서비스만
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --zone=trusted --list-all
```

👀 **보이는 것:** trusted에 소스 대역이, public에 https만. 내부망에선 다 되고 외부에선 https만.

💡 **느낄 것:** 규칙을 포트 하나하나가 아니라 **"이 대역은 믿는다/안 믿는다"** 로 생각하게 된다. ufw(데비안계)와 firewalld(레드햇계)는 사고방식이 다르다.
  단, Docker publish 포트엔 zone 규칙이 안 먹는다 — 그건 FORWARD로 가니 **egress=docker인 policy**로, 호스트 포트가 아니라 **컨테이너 포트** 기준으로 막는다(③의 ufw 함정과 같은 뿌리).

📖 심화: `ref/docker-firewalld-zones.html` — zone vs policy, `daemon.json` 대역 겹침.

