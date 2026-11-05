## 실습⑪ 프라이빗 CA — 내가 신뢰의 뿌리가 된다
🎯 **목표:** 자체 CA로 서버 인증서를 찍고, **윈도·폰이 그걸 믿게** 만들고, mTLS까지.
🖥️ **랩:** 작업 VM + 참가자 폰 + (가능하면) 윈도 노트북 or lubuntu VM 브라우저.

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
👀 서버에 `srv.crt` 물리고 브라우저로 열면 → **⚠️ 경고**(아무도 이 CA를 모름).

⌨️ **해보기 — (2) 윈도·폰 키스토어에 CA 심기**
```
윈도: ca.crt 더블클릭 → 인증서 설치 → "로컬 컴퓨터" →
      "신뢰할 수 있는 루트 인증 기관"에 저장
iOS : ca.crt 전송 → 설정>프로파일 설치 → 설정>일반>정보>
      인증서 신뢰 설정에서 "전체 신뢰" 토글 ON
안드: 설정>보안>암호화 및 자격증명>CA 인증서 설치
```
👀 **보이는 것:** CA 심기 **전** 경고 → 심은 **후** 같은 사이트가 자물쇠 🔒 정상.
💡 **느낄 것:** 공인 CA가 하는 일을 내가 똑같이 했다. "신뢰"는 마법이 아니라 **키스토어에 뿌리 인증서가 있느냐** 한 줄 문제. 사설 CA면 그 뿌리를 쓰는 기기마다 **내가 직접 심어야** 한다(=관리 비용).
💡 **실감 훅 — 회사 DPI 장비:** 사내 TLS 복호화(DPI) 장비가 당신 HTTPS를 **중간에서 풀어 보고 자기 CA로 다시 서명**해 돌려준다. 가능한 이유? 그 CA가 **당신 OS 키체인에 이미 심겨 있어서** — 방금 (2)에서 당신이 한 바로 그 동작. 즉 **"CA 신뢰 = 그 주체에게 내 트래픽 열람권을 준 것"**. 같은 칼이 보호도 되고 감시도 된다. (`danawa.com` 인증서를 사내망 안/밖에서 비교하면 발급자가 다름.)

⌨️ **해보기 — (3) mTLS: 서버도 클라이언트를 검사**
```bash
# 클라이언트 인증서도 같은 CA로 발급
openssl genrsa -out cli.key 2048
openssl req -new -key cli.key -subj "/CN=⟨alice⟩" -out cli.csr
openssl x509 -req -in cli.csr -CA ca.crt -CAkey ca.key -days 365 -out cli.crt
# caddy에 client_auth(require_and_verify) 설정 후
curl --cacert ca.crt https://⟨srv.lab.local⟩           # ❌ 403 (클라 인증서 없음)
curl --cacert ca.crt --cert cli.crt --key cli.key \
     https://⟨srv.lab.local⟩                           # ✅ 통과
```
👀 **보이는 것:** 인증서 없는 curl은 거부(TLS handshake 실패/403), 클라 인증서 붙이면 통과.
💡 **느낄 것:** 평소 HTTPS는 **서버만** 신분증을 보인다. mTLS는 **양쪽이** 신분증을 보인다. 비번·토큰 없이 "인증서 가진 기기만" 접속 — 관리망·API에 강력. (폰에 cli 인증서 넣어서 브라우저로도 체험)

