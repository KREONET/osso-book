## 실습⑥ 나가는 연결 제한 — egress 화이트리스팅 {#lab-06}

🎯 **목표:** 아웃바운드를 기본 차단하고 squid로 꼭 필요한 곳만 연다. 막힌 시도를 **로그로 찾고**, 필요할 때 **잠깐 풀었다 다시 잠그는** 손놀림까지 익힌다.

🖥️ **랩:** 작업 VM. squid 프록시(클라이언트 대신 외부에 접속해 주는 중계 서버)는 랩에 미리 실행되어 있다(allowlist = apt·yum·docker·ghcr).

⌨️ **1단계 — 잠그기**
```bash
# 나가는 걸 다 막는다
sudo ufw default deny outgoing
# squid만 예외
sudo ufw allow out to ⟨203.0.113.9⟩ port 3128
# DNS
sudo ufw allow out 53
# 막힌 시도를 로그로 남긴다
sudo ufw logging low
```

⌨️ **2단계 — apt와 Docker가 프록시를 보게 한다.** 둘 다 ①에서 배운 드롭인(`*.d/`에 조각 파일을 얹는 방식)이다.
```bash
# apt: apt.conf.d 에 조각 하나
echo 'Acquire::http::Proxy "http://⟨203.0.113.9⟩:3128";' | sudo tee /etc/apt/apt.conf.d/01proxy
# Docker 데몬: 서비스 드롭인 디렉터리에 조각 하나
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf <<'EOF'
[Service]
Environment="HTTP_PROXY=http://⟨203.0.113.9⟩:3128"
Environment="HTTPS_PROXY=http://⟨203.0.113.9⟩:3128"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

⌨️ **3단계 — 시험.** 줄마다 성공할지 먼저 예측해 보자.
```bash
# allowlist 안의 저장소
sudo apt update
docker pull ghcr.io/⟨org/img⟩
# allowlist 밖의 사이트 — 직접
curl -I -m 5 https://example.com
# 같은 사이트 — 이번엔 프록시를 거쳐서
curl -I -m 5 -x http://⟨203.0.113.9⟩:3128 https://example.com
```

⌨️ **4단계 — 무엇이 막혔나, 로그로 찾는다**

너무 조이면 반드시 무언가가 안 되는 순간이 온다. 그때 짐작으로 풀지 말고 **막힌 기록**을 본다.
```bash
# 직접 나가려다 막힌 시도 — 나가는 패킷은 'IN= OUT=장치' 로 찍힌다
sudo journalctl -k | grep 'UFW BLOCK' | grep 'IN= OUT='
# 어디로 나가려 했나 — 목적지:포트별 횟수
sudo journalctl -k | grep 'UFW BLOCK' | grep 'IN= OUT=' \
  | sed -nE 's/.*DST=([^ ]+).*DPT=([0-9]+).*/\1:\2/p' | sort | uniq -c | sort -rn | head
```

> ufw 로그는 1분에 3건 정도만 **표본으로** 남긴다(로그 폭주 방지). 모든 시도가 기록되지는 않으므로 "어디로 가려 했나"를 찾는 용도로 쓴다. 프록시가 거절한 요청(allowlist 밖 도메인)은 squid 서버의 `/var/log/squid/access.log`에 `TCP_DENIED`로 남는다.

⌨️ **5단계 — 토글: 잠깐 풀었다 다시 잠그기**

셸 설정 파일(`~/.bashrc` 또는 `~/.zshrc`)에 넣어 두면 편하다.
```bash
egress-open()  { sudo ufw default allow outgoing && echo "⚠️ egress 열림 — 끝나면 egress-close"; }
egress-close() { sudo ufw default deny outgoing; }
egress-state() { sudo ufw status verbose | grep -i '^default'; }
```

```bash
egress-open
egress-state
# … 필요한 작업 …
egress-close
egress-state
```

> 전체를 열기보다 **필요한 목적지만 잠깐** 여는 편이 낫다: `sudo ufw allow out to ⟨IP⟩ port 443` → 끝나면 `sudo ufw delete allow out to ⟨IP⟩ port 443`. 4단계 로그에서 찾은 목적지를 그대로 쓰면 된다. 자주 필요한 곳이라면 토글 대신 squid allowlist에 올리는 것이 맞다.

👀 **보이는 것:**
- `apt update`와 `docker pull`은 성공한다(프록시 경유, allowlist 안).
- 직접 `curl`은 5초 뒤 타임아웃. 프록시를 거친 `curl`은 `403 Forbidden`으로 즉시 거절된다. 막는 주체(방화벽과 프록시)가 다르다.
- 4단계 로그에 `DST=⟨example.com의 IP⟩ … DPT=443`이 찍혀 있다.
- `egress-open` 뒤 `egress-state`는 `allow (outgoing)`, `egress-close` 뒤엔 `deny (outgoing)`.

💡 **느낄 것:** 인바운드만 막던 습관에서 벗어난다. **나가는 쪽을 화이트리스팅**(허용 목록에 있는 곳만 통과시키는 방식)하면 서버가 침해되어도 공격자가 밖으로 나갈 길이 없다. 또 조이는 것만큼 **조인 상태로 운영하는 법**도 중요하다. 막히면 로그로 목적지를 찾고, 필요한 만큼만 잠깐 열고, 다시 잠근다. 이 습관이 없으면 결국 누군가 불편하다며 전부 풀어 버리고 다시 잠그지 않는다. Day2에서는 같은 통제를 AI에 적용한다.

⚠️ `ufw default deny outgoing`은 **호스트가 시작한** 연결만 막는다. 컨테이너가 시작한 연결은 FORWARD로 빠져서 이 규칙에 닿지 않는다 → [⑨′](lab-09p.md)에서 다룬다.

> 💭 **더 해 보고 싶다면 — 풀어 놓고 잊었다면?** `egress-open`으로 열어 놓고 다시 잠그는 것을 잊는 일이 가장 흔한 사고다. 열면 30분 뒤 **자동으로** 잠기는 장치를 systemd 타이머로 만들 수 있다. 관심이 있으면 AI에게 맡겨 보자.
>
> ```text
> 우분투 서버에서 ufw로 아웃바운드를 기본 차단(default deny outgoing)해 두었다.
> 작업 때문에 가끔 잠깐 풀어야 하는데, 다시 잠그는 걸 잊을까 걱정이다.
> `egress-open 30m`처럼 풀면 정한 시간 뒤 자동으로 다시 잠기는 장치를 systemd로 만들어 줘.
> - 열 때 누가·언제·왜(사유 한 줄)를 journal에 남길 것
> - 시간이 되기 전에 직접 닫으면 예약도 취소할 것
> - 재부팅해도 '열린 채로' 남지 않을 것
> - 파일을 만들기 전에 설계부터 보여 주고, 내가 동의하면 만들 것
> - 테스트 VM에서 검증하는 절차도 함께
> ```

---

🔀 **갈림길** — 다음엔 어디로?

- 같은 egress를 컨테이너 고도에서 → [⑨′ 컨테이너 egress](lab-09p.md)
- 같은 egress를 망 경계 고도에서 → [⑧ 망 간 접점·bastion](lab-08.md)
- 격리망 자체를 만들어 보기 → [⑦ 방화벽·VPN](lab-07.md)
- 왜 아웃바운드가 더 중요한가 → [Part 2 §2.6](../02-firewall.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
