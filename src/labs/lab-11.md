## 실습⑪ 프라이빗 CA — 직접 신뢰의 뿌리 만들기 {#lab-11}

🎯 **목표:** 자체 CA로 서버 인증서를 발급하고, **윈도·휴대전화가 그 인증서를 신뢰하게** 만들고, mTLS(서버와 클라이언트가 서로 인증서를 확인하는 방식)까지 해 본다.

🖥️ **랩:** 작업 VM + 참가자 휴대전화 + (가능하면) 윈도 노트북 또는 lubuntu VM 브라우저.

⌨️ **해보기 — (1) CA와 서버 인증서 만들기**
```bash
# 루트 CA (내 조직의 신뢰 뿌리)
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -days 3650 -out ca.crt \
  -subj "/CN=⟨MyLab Root CA⟩"
# 서버 인증서를 CA로 서명
openssl genrsa -out srv.key 2048
openssl req -new -key srv.key -subj "/CN=⟨srv.lab.local⟩" -out srv.csr
openssl x509 -req -in srv.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 365 -out srv.crt -extfile <(printf "subjectAltName=DNS:⟨srv.lab.local⟩")
```

👀 서버에 `srv.crt`를 적용하고 브라우저로 열면 → **⚠️ 경고**(이 CA를 아는 곳이 없음).

⌨️ **해보기 — (2) 윈도·휴대전화 키스토어에 CA 넣기**
```
윈도: ca.crt 더블클릭 → 인증서 설치 → "로컬 컴퓨터" →
      "신뢰할 수 있는 루트 인증 기관"에 저장
iOS : ca.crt 전송 → 설정>프로파일 설치 → 설정>일반>정보>
      인증서 신뢰 설정에서 "전체 신뢰" 토글 ON
안드: 설정>보안>암호화 및 자격증명>CA 인증서 설치
```

👀 **보이는 것:** CA를 키스토어(기기가 신뢰하는 인증서를 보관하는 저장소)에 넣기 **전**에는 경고 → 넣은 **후**에는 같은 사이트가 자물쇠 🔒로 정상 표시.

💡 **느낄 것:** 공인 CA가 하는 일을 내가 그대로 해 보았다. 신뢰는 결국 **키스토어에 뿌리 인증서가 있는지**의 문제다. 사설 CA라면 그 뿌리 인증서를 쓰는 기기마다 **직접 넣어야** 하고, 이것이 관리 비용이 된다.

💡 **현실의 예 — 회사 DPI 장비:** 사내 TLS 복호화(DPI, 패킷 내용까지 들여다보는 검사) 장비는 사용자의 HTTPS를 **중간에서 풀어 보고 자기 CA로 다시 서명**해 돌려준다. 이것이 가능한 까닭은 그 CA가 이미 사용자 OS의 키체인에 들어 있기 때문이다. 방금 (2)에서 직접 한 것과 같은 동작이다. 어떤 CA를 신뢰한다는 것은 **그 주체에게 내 트래픽을 열어 볼 권한을 준다**는 뜻이다. 같은 도구가 보호에도 감시에도 쓰인다. (같은 HTTPS 사이트, 예컨대 `example.com`의 인증서를 사내망 안팎에서 비교하면 발급자가 다르다. [§5.5](../05-trust.md) 참고.)

⌨️ **해보기 — (3) mTLS: 서버도 클라이언트를 검사**
```bash
# 클라이언트 인증서도 같은 CA로 발급
openssl genrsa -out cli.key 2048
openssl req -new -key cli.key -subj "/CN=⟨alice⟩" -out cli.csr
openssl x509 -req -in cli.csr -CA ca.crt -CAkey ca.key -days 365 -out cli.crt
# caddy에 client_auth(require_and_verify) 설정 후
# ❌ 403 (클라 인증서 없음)
curl --cacert ca.crt https://⟨srv.lab.local⟩
# ✅ 통과
curl --cacert ca.crt --cert cli.crt --key cli.key \
     https://⟨srv.lab.local⟩
```

👀 **보이는 것:** 인증서 없는 curl은 거부(TLS handshake 실패/403), 클라이언트 인증서를 붙이면 통과.

💡 **느낄 것:** 평소 HTTPS는 **서버만** 신분증을 보인다. mTLS는 **양쪽이** 신분증을 보인다. 비밀번호·토큰 없이 인증서를 가진 기기만 접속할 수 있어 관리망·API 보호에 효과적이다. 이것이 **제로 트러스트**다. 어느 망에서 왔는지보다 무엇을 들고 왔는지로 판단한다. 인증서의 CN에 조건을 걸면 사람·팀마다 권한을 나눌 수도 있다([Part 5 §5.6](../05-trust.md)). (휴대전화에 cli 인증서를 넣어 브라우저로도 확인해 볼 수 있다.)

---

🔀 **갈림길** — 다음엔 어디로?

- 앱 앞에 로그인 문지기 세우기 → [⑫ OIDC forward-auth](lab-12.md)
- 신뢰 체인과 키 보관 → [Part 5](../05-trust.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
