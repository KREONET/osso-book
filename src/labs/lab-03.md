## 실습③ ufw를 켰는데 왜 뚫리지? — Docker 방화벽 우회

🎯 **목표:** Docker가 ufw를 건너뛰는 함정을 눈으로 보고 잡는다.

🖥️ **랩:** 작업 VM.

⌨️ **해보기**
```bash
sudo ufw enable
sudo ufw deny 80/tcp
docker run -d -p 80:80 --name web nginx
curl -I http://localhost          # ← 어? 200이 온다 (막았는데!)
# 해결
sudo ufw-docker install
sudo ufw route delete allow proto tcp from any to any port 80 2>/dev/null
curl -I http://localhost          # ← 이제 막힌다
```

👀 **보이는 것:** deny 후에도 첫 curl은 `HTTP/1.1 200 OK`. ufw-docker 적용 후 연결 거부/타임아웃.
  - 어느 체인이 처리하나: `sudo iptables-nft -L DOCKER-USER -v -n` (카운터) · `journalctl -k | grep "UFW DOCKER BLOCK"`

💡 **느낄 것:** "방화벽 켰으니 안전"이 **거짓**일 수 있다. publish 포트는 `nat PREROUTING`에서 DNAT으로 목적지가 컨테이너로 바뀌어 **INPUT이 아니라 FORWARD로 흐른다** — ufw는 INPUT만 지키니 질문조차 받지 못한다. 그래서 `ufw status`에 80이 없어도 열려 있다. → `ufw-docker`가 `DOCKER-USER` 체인을 채워 진짜 해결.

⚠️ 기본 `after.rules`는 출발지 10/8·172.16/12·192.168/16을 무조건 통과시킨다 — 클라이언트가 사설망이면 여전히 열린 셈. 엄격히 하려면 그 줄을 좁힌다.

📖 심화: `ref/docker-ufw-nftables.html` — nft 체인 3단계 추적.

