## 실습⑩ 진짜 인증서 — HTTP-01 vs DNS-01 두 챌린지

🎯 **목표:** Let's Encrypt가 "이 도메인 네 것 맞아?"를 확인하는 **두 방식**을 몸으로 겪는다.

🖥️ **랩:** 작업 VM(caddy). 참가자별 `eNN` 서브도메인 사전 위임됨.

⌨️ **해보기**
```bash
# A) HTTP-01 — 80포트로 챌린지 파일을 되돌려준다 (공개 서버여야 함)
caddy reverse-proxy --from ⟨⟨svc.demo.example⟩⟩ --to localhost:8080
#   → caddy가 자동으로 인증서 발급, 브라우저 자물쇠 확인

# B) DNS-01 — 80 안 열고, DNS TXT 레코드로 증명 (내부/와일드카드 가능)
#   acme-dns에 _acme-challenge.eNN 만 위임받았으므로 존 전체 권한 없이 발급
export ACME_DNS_API=⟨https://⟨auth.acme-dns.example⟩⟩
# dns 챌린지 프로바이더 설정됨
caddy run --config Caddyfile.dns01
```

👀 **보이는 것:** A는 80포트 요청 로그 후 발급. B는 포트 안 열고도 `certificate obtained` — DNS에 TXT가 잠깐 생겼다 사라짐.

💡 **느낄 것:** HTTP-01은 "문을 열어 보여주기", DNS-01은 "등기부로 증명하기". **내부망·와일드카드엔 DNS-01**. 그리고 `_acme-challenge`만 위임하면 **존 전체를 안 주고도** 각자 발급받는다(최소권한).

---

🔀 **갈림길** — 다음엔 어디로?

- 내부망이라 공인 인증서가 닿지 않는다면 → [⑪ 프라이빗 CA·mTLS](lab-11.md)
- 이 앞에 로그인을 붙이기 → [⑫ OIDC forward-auth](lab-12.md)
- ACME·인증서 체인의 원리 → [Part 5](../05-trust.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
