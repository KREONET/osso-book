## 실습③ ufw를 켰는데 왜 뚫리지? — Docker 방화벽 우회

🎯 **목표:** Docker가 ufw를 건너뛰는 함정을 **밖에서** 눈으로 보고, 고친 줄 알았던 것까지 다시 의심해 본다.

🖥️ **랩:** 작업 VM 2대 — **내 VM**(서버)과 **짝의 VM**(밖에서 두드리는 쪽). 옆자리와 짝을 짓고 서로의 IP를 알려 준다. 짝이 없으면 강사 VM이나 lubuntu VM에서 두드린다.

> ⚠️ **방화벽은 반드시 다른 머신에서 시험한다.** 같은 머신에서 `curl localhost`를 하면 패킷이 루프백(`lo`)으로만 돌아서 방화벽 규칙을 거치지 않는다. ufw는 `lo`를 기본으로 허용하므로, 방화벽이 있든 없든 200이 나온다. 그 결과로는 아무것도 증명되지 않는다.

⌨️ **준비 — 서로의 IP 확인** (두 VM 모두)
```bash
hostname -I
```

⌨️ **1단계 — 막았는데 열려 있다** (내 VM)
```bash
sudo ufw enable
sudo ufw deny 80/tcp
docker run -d -p 80:80 --name web nginx
```

(짝의 VM에서)
```bash
# 예측: 80을 deny 했다. 열릴까, 막힐까?
curl -I -m 3 http://⟨내 VM IP⟩
```

⌨️ **2단계 — ufw-docker로 고친다** (내 VM)
```bash
sudo ufw-docker install
sudo ufw reload
```

(짝의 VM에서)
```bash
# 예측: 이제는 막혔을까?
curl -I -m 3 http://⟨내 VM IP⟩
```

⌨️ **3단계 — 왜 아직 열려 있나, 직접 찾는다** (내 VM)
```bash
# ufw-docker가 넣은 규칙에서 '출발지가 이 대역이면 그냥 통과(RETURN)' 줄을 찾는다
grep -n 'RETURN -s' /etc/ufw/after.rules
# 실제 커널 체인에도 같은 줄이 있다
sudo iptables-nft -L DOCKER-USER -n --line-numbers
```

> `iptables-nft`는 iptables **문법**으로 보여 주지만, 규칙은 실제로 nftables에 들어 있다. Ubuntu 22.04·Debian 10·RHEL/Rocky 8 이후에는 그냥 `iptables`도 이 명령이다(iptables 문법 → nftables로 번역). 언제부터 바뀌었는지는 [Part 2 §2.1](../02-firewall.md)에서.

교육장 VM들은 사설 IP(`10.x`·`172.16~31.x`·`192.168.x`)를 쓴다. 짝의 IP가 그 대역 안에 있는지 보라. 그렇다면 그 세 줄을 **주석 처리**한다. 지우지 않고 주석으로 두면 나중에 되돌리기 쉽다.
```bash
# 편집기로 열어 RETURN -s 10.0.0.0/8 · 172.16.0.0/12 · 192.168.0.0/16 세 줄 맨 앞에 # 을 붙인다
sudo nano /etc/ufw/after.rules
sudo ufw reload
# 세 줄이 체인에서 빠졌는지 확인
sudo iptables-nft -L DOCKER-USER -n --line-numbers
```

(짝의 VM에서)
```bash
# 예측: 이번엔?
curl -I -m 3 http://⟨내 VM IP⟩
```

⌨️ **4단계 — 필요한 것만 연다** (내 VM)
```bash
sudo ufw-docker allow web 80/tcp
```

(짝의 VM에서)
```bash
curl -I -m 3 http://⟨내 VM IP⟩
```

👀 **보이는 것:**
- 1단계: `deny`했는데도 `HTTP/1.1 200 OK`. `sudo ufw status`에는 80이 막혀 있다고 나온다.
- 2단계: ufw-docker를 깔았는데도 **여전히 200.** 짝이 사설망에 있기 때문이다.
- 3단계: 세 줄을 빼자 `curl: (28) Connection timed out` — 이제야 막힌다.
- 4단계: 명시적으로 허용한 `web`의 80만 다시 200.
- 막힌 시도 기록: `sudo journalctl -k | grep "UFW DOCKER BLOCK"`. 어느 규칙이 처리했는지: `sudo iptables-nft -L DOCKER-USER -v -n`의 카운터.

💡 **느낄 것:** "방화벽 켰으니 안전"이 **거짓**일 수 있다. publish 포트는 `nat PREROUTING`에서 DNAT으로 목적지가 컨테이너로 바뀌어 **INPUT이 아니라 FORWARD로 흐른다.** ufw는 INPUT만 지키니 이 패킷은 질문조차 받지 않는다. 그리고 **도구를 깔았다고 끝이 아니다.** ufw-docker도 기본값으로 사설망을 통째로 믿는다. 기관 내부망이 바로 그 사설망이다. 고친 다음엔 반드시 **밖에서, 실제 클라이언트 위치에서** 다시 두드려 보고, 기본 예외 규칙을 직접 읽는다. 끝 상태는 "기본 차단 + 필요한 것만 명시적 허용"이다.

⚠️ `ufw-docker allow`는 그 순간의 **컨테이너 IP**에 규칙을 묶는다. 컨테이너를 다시 만들어 IP가 바뀌면 허용도 다시 해야 한다.

---

🔀 **갈림길** — 다음엔 어디로?

- RHEL·Rocky 계열이라 firewalld를 쓴다면 → [④ firewalld](lab-04.md)
- 들어오는 쪽을 평판으로 미리 거르기 → [⑤ 인바운드 선차단](lab-05.md)
- 왜 우회되나 — DNAT·FORWARD·DOCKER-USER 체인 → [Part 2](../02-firewall.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
