## 실습⑫ 로그인은 앱 말고 문지기에게 — OIDC forward-auth
🎯 **목표:** 앱에 로그인 기능이 없어도 **앞단 프록시**가 인증을 대신 받게.
🖥️ **랩:** 작업 VM(oauth2-proxy + 자체 IdP 사전 구성, 혹은 Auth0 테넌트).

⌨️ **해보기**
```bash
cat compose.yaml            # oauth2-proxy 가 upstream 앞에 선다
docker compose up -d
# 보호 안 된 앱은 그냥 열림. 보호된 경로는 프록시가 가로챔.
curl -I https://⟨app.lab.local⟩/private     # → 302 로그인 리다이렉트
```
그리고 브라우저로 `https://⟨app.lab.local⟩/private` 접속 → IdP 로그인 화면 → 로그인하면 통과.
👀 **보이는 것:** 앱은 인증 코드 한 줄 없는데, `/private`가 로그인 화면으로 튕김. 로그인 후 헤더에 사용자 정보(`X-Auth-Request-User`) 주입.
💡 **느낄 것:** 인증을 **앱마다 짜지 않는다**. 문지기(forward-auth) 하나가 모든 앱 앞을 지키고, 앱은 "누가 들어왔는지"만 헤더로 받는다. 섀도우 IT 앱도 이걸 앞에 세우면 **로그인이 생긴다**.

