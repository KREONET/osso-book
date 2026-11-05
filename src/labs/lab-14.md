## 실습⑭ 〔캡스톤〕AI에게 배포를 맡기되 — 시크릿·egress로 가둔다

🎯 **목표:** AI가 앱을 배포하게 하되, ①시크릿을 **안 읽히고** 쓰고 ②egress로 **못 빠져나가게**.

🖥️ **랩:** 작업 VM(⑥의 egress 화이트리스팅 + squid 재사용).

⌨️ **해보기 — (1) 시크릿은 주되 LLM 눈엔 안 보이게**
```bash
# .env 는 compose가 읽고 컨테이너에 주입. LLM에겐 "있다"만 알림.
printf 'DB_PASSWORD=⟨s3cr3t⟩\nAPI_KEY=⟨xxxx⟩\n' > .env
chmod 600 .env
# Claude Code 설정: .env 를 Read 거부 목록에
# "deny": ["Read(./.env)"] 사전 세팅됨
cat .claude/settings.json
# 이제 AI에게: "compose로 앱 배포해줘. .env 는 열어보지 말고 그대로 쓰면 돼."
```

👀 AI는 `docker compose up`을 실행하고 앱은 비번으로 DB에 붙는데, **AI 대화엔 비번이 한 번도 안 나온다**. `.env`를 열려 하면 **거부**됨.

⌨️ **해보기 — (2) 뚫려도 못 나가게: egress로 AI 작업 가두기**
```bash
# 이 VM은 ⑥에서 이미 아웃바운드 기본 차단 + squid(apt/ghcr만).
# AI가 임의 사이트로 데이터를 빼내려 해도:
# ❌ 타임아웃 (egress 차단)
docker exec app curl -I https://⟨evil.example⟩
# ✅ (allowlist)
docker exec app curl -I https://ghcr.io
```

👀 **보이는 것:** 허용된 레지스트리/저장소만 나가고 나머지는 막힘. AI든 앱이든 **구멍이 생겨도 데이터가 못 샌다**.

💡 **느낄 것:** AI에게 서버를 맡기는 게 겁나는 이유는 "뭘 들고 튈까"다. 답은 신뢰가 아니라 **경계** — 시크릿은 손잡이로만 쥐여주고, 나가는 길은 화이트리스팅으로 막는다. 이게 **"맡기되 안전하게"의 실체**. Day1에서 깐 egress가 여기서 AI 통제장치가 된다.

---

🔀 **갈림길** — 다음엔 어디로?

- 전체를 꿰는 그림 — SADP 4통제지점 → [Part 7](../07-ai-governance.md)
- 배운 걸 우리 연구실에 써먹는 과제 만들기 → [AI로 내 교재 만들기: 우리 환경 과제](../ai/my-task.md)
- 다른 줄기를 골라 다시 → [🗺️ 지도](../map.md)
