## 실습⑫ 로그인을 앞단 프록시에 맡기기 — OIDC forward-auth {#lab-12}

🎯 **목표:** 앱에 로그인 기능이 없어도 **앞단 프록시**가 OIDC(OpenID Connect, 로그인을 외부 인증 서버에 맡기는 표준 규약)로 인증을 대신 처리하게 한다.

🖥️ **랩:** 작업 VM(oauth2-proxy + 자체 IdP(신원 제공자, 로그인을 처리하는 인증 서버) 사전 구성, 혹은 Auth0 테넌트).

⌨️ **해보기**
```bash
# oauth2-proxy 가 upstream 앞에 선다
cat compose.yaml
docker compose up -d
# 보호 안 된 앱은 그냥 열림. 보호된 경로는 프록시가 가로챔.
# → 302 로그인 리다이렉트
curl -I https://⟨app.lab.local⟩/private
```
그리고 브라우저로 `https://⟨app.lab.local⟩/private` 접속 → IdP 로그인 화면 → 로그인하면 통과.

👀 **보이는 것:** 앱에는 인증 코드가 한 줄도 없는데 `/private`에 접속하면 로그인 화면으로 넘어감. 로그인 후 헤더에 사용자 정보(`X-Auth-Request-User`) 주입.

💡 **느낄 것:** 인증을 **앱마다 구현하지 않는다**. 문지기(forward-auth) 하나가 모든 앱 앞을 지키고, 앱은 "누가 들어왔는지"만 헤더로 받는다. 섀도우 IT 앱도 이것을 앞에 두면 **로그인이 생긴다**.

---

🔀 **갈림길** — 다음엔 어디로?

- AI에게 서버를 맡겨 보기 → [⑬ AI에게 서버 조사시키기](lab-13.md)
- SSO·OIDC·접근 통제 → [Part 6](../06-access.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
