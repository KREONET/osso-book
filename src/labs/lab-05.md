## 실습⑤ 인바운드 선차단 — 평판 나쁜 IP는 노크도 못 하게

🎯 **목표:** 평판 기반 차단을 ufw/fail2ban **앞단**에 둔다.

🖥️ **랩:** 작업 VM(republock 설치됨).

⌨️ **해보기**
```bash
sudo republock enable spammers abuseipdb asndrop
# 약 11만 엔트리 로드
sudo republock status
sudo nft list set inet republock blocklist | head
# 로그에서 차단되는 것 관찰
sudo journalctl -k | grep -i drop | tail
```

👀 **보이는 것:** status에 그룹별 엔트리 수. 커널 nftables 세트에 수만 prefix. 포트 스캔/노크 로그가 눈에 띄게 준다.

💡 **느낄 것:** fail2ban이 "들어온 뒤 벤"이라면 이건 **들어오기 전 드랍**. 알려진 쓰레기 11만개를 커널이 먼저 쳐낸다.

---

🔀 **갈림길** — 다음엔 어디로?

- 이번엔 나가는 쪽을 막기 → [⑥ egress 화이트리스팅](lab-06.md)
- 인·아웃바운드를 함께 보는 그림 → [Part 2](../02-firewall.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
