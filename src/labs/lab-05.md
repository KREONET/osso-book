## 실습⑤ 인바운드 선차단 — 평판 나쁜 IP는 노크도 못 하게

🎯 **목표:** 평판 기반 차단을 ufw·fail2ban **앞단**에 두고, 앞단이 먼저 막는다는 걸 짝의 패킷으로 직접 확인한다. 국가별 IP 목록이 어디서 오는지도 직접 뽑아 본다.

🖥️ **랩:** 작업 VM 2대 — **내 VM**([republock](https://github.com/KREONET/republock) 설치됨)과 **짝의 VM**. ③의 짝 그대로.

> 🧑‍🏫 강사 준비: 랩 VM에 republock을 설치할 때 `ADMIN_NETS`에는 **강사 관리망만** 넣는다. 교육장 대역을 넣으면 그 대역은 무조건 통과라서 3단계에서 짝이 막히지 않는다.

⌨️ **1단계 — 둘러보기** (내 VM)
```bash
# 그룹별 상태와 목록 규모
sudo republock status
# 커널에 올라간 실제 목록 — 그룹마다 set 하나
sudo nft list set inet republock spammers | head -20
# 어느 순서로 검사되나: 숫자가 작을수록 먼저
sudo nft list ruleset | grep -E '^table|hook'
```

⌨️ **2단계 — 국가별 IP 목록은 어디서 오나** (내 VM)

IP 주소는 지역 인터넷 주소 관리 기구(아시아·태평양은 APNIC)가 나눠 주고, **어느 대역을 어느 나라에 줬는지 매일 공개**한다.
```bash
curl -s https://ftp.apnic.net/stats/apnic/delegated-apnic-extended-latest -o apnic.txt
# 형식: 기구|국가|종류|시작주소|주소 개수|할당일|상태
grep '|KR|ipv4|' apnic.txt | head -3
# 한국에 할당된 IPv4 블록 수
grep -c '|KR|ipv4|' apnic.txt
```

출력 예 (2026-11-05):
```text
apnic|KR|ipv4|1.11.0.0|65536|20100616|allocated|A922C04D
apnic|KR|ipv4|1.16.0.0|16384|20100512|allocated|A9222B0E
apnic|KR|ipv4|1.16.64.0|16384|20100512|allocated|A92E438A
2422
```

⌨️ **3단계 — 짝을 블록리스트에 올린다**

(짝의 VM에서, 먼저)
```bash
# 예측: ufw는 켜져 있다. ping이 갈까?
ping -c 3 -W 1 ⟨내 VM IP⟩
```

(내 VM)
```bash
# 짝의 IP 하나짜리 목록 파일
echo '⟨짝 VM IP⟩/32' | sudo tee /etc/republock/lab.txt
# 설정 끝에 연습용 'lab' 그룹을 덧붙인다 (설정 파일에서는 나중 줄이 앞 줄을 덮는다)
sudo tee -a /etc/republock/republock.conf <<'EOF'
# 실습⑤: 짝 VM을 막아 보는 연습용 그룹
EXTRA_GROUPS="spammers abuseipdb asndrop lab"
SRC4[lab]="file:///etc/republock/lab.txt"
EOF
sudo republock update
sudo republock check ⟨짝 VM IP⟩
```

(짝의 VM에서)
```bash
# 예측: 이번엔?
ping -c 3 -W 1 ⟨내 VM IP⟩
```

⌨️ **4단계 — 풀기** (내 VM)
```bash
# 규칙만 빼고 목록은 남긴다
sudo republock disable lab
```
짝이 다시 ping을 보내 본다.

👀 **보이는 것:**
- 1단계: `spammers`·`abuseipdb`·`asndrop` 세 그룹 합쳐 약 11만 개 대역. `table inet republock`이 `priority -150`, ufw 쪽이 `priority 0`.
- 2단계: 한국 IPv4 블록이 2천 개 넘게 나온다. 국가별 IP 목록은 비밀이 아니라 공개 장부다.
- 3단계: 처음엔 ping이 간다(ufw는 ping을 기본 허용). 짝을 올린 뒤에는 `100% packet loss`. `republock check`는 `BLOCKED by group:lab`.
- 4단계: `disable` 하자 ping이 바로 돌아온다.

💡 **느낄 것:** ufw는 ping을 허용하는데도 짝은 막혔다. **앞단(-150)이 먼저 보고 버렸기 때문에** ufw는 그 패킷을 구경조차 못 했다. fail2ban이 "들어와서 나쁜 짓을 한 뒤 벤"이라면 이건 **노크 전에 드랍**이다. 알려진 쓰레기 11만 개를 커널이 먼저 쳐내니 뒤의 방화벽과 로그가 깨끗해진다. 그리고 국가 차단은 공개 장부 덕에 **쉽다** — 쉬운 만큼 그 나라의 정상 사용자까지 다 막는다는 것도 함께 기억하자.

⚠️ 차단은 **조용하다.** 로그도 응답도 없다. "왜 안 붙지?" 싶으면 먼저 `sudo republock check ⟨IP⟩`.

⚠️ 국가 차단을 켠 서버라면 **그 나라로 출장 가기 전에** `sudo republock disable ⟨국가⟩`. 도착한 뒤엔 내 IP가 이미 막혀 있어 원격으로 풀 수 없다.

> 💡 **현장 활용:** 유지보수가 끝나 규칙 갱신이 번거로운 오래된 방화벽 앞에, 10G NIC 두 장 꽂은 PC로 republock **투명 브릿지**를 끼우면 평판 목록이 매주 알아서 갱신된다 → [Part 2 §2.5](../02-firewall.md).

---

🔀 **갈림길** — 다음엔 어디로?

- 이번엔 나가는 쪽을 막기 → [⑥ egress 화이트리스팅](lab-06.md)
- 목록은 누가 만드나, 국가 차단의 한계, 방화벽 앞 투명 브릿지 → [Part 2 §2.5](../02-firewall.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
