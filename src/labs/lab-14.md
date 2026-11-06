## 실습⑭ 〔캡스톤〕AI에게 배포 맡기기 — 시크릿 분리와 egress 격리 {#lab-14}

🎯 **목표:** AI가 앱을 배포하게 하되, ① 시크릿은 AI가 **읽지 못하게** 한 채 쓰게 하고 ② egress로 **밖으로 빠져나가지 못하게** 한다.

🖥️ **랩:** 작업 VM(⑥의 egress 화이트리스팅 + squid 재사용).

⌨️ **해보기 — (1) 시크릿은 주되 LLM에는 보이지 않게**
```bash
# .env 는 compose가 읽고 컨테이너에 주입. LLM에겐 "있다"만 알림.
printf 'DB_PASSWORD=⟨s3cr3t⟩\nAPI_KEY=⟨xxxx⟩\n' > .env
chmod 600 .env
# Claude Code 설정: .env 를 Read 거부 목록에
# "deny": ["Read(./.env)"] 사전 세팅됨
cat .claude/settings.json
# 이제 AI에게: "compose로 앱 배포해줘. .env 는 열어보지 말고 그대로 쓰면 돼."
```

👀 AI는 `docker compose up`을 실행하고 앱은 비밀번호로 DB에 접속하는데, **AI 대화에는 비밀번호가 한 번도 나오지 않는다**. `.env`를 열려 하면 **거부**됨.

⌨️ **해보기 — (2) 침해되어도 나가지 못하게: egress로 AI 작업 가두기**
```bash
# 이 VM은 ⑥에서 이미 아웃바운드 기본 차단 + squid(apt/ghcr만).
# AI가 임의 사이트로 데이터를 빼내려 해도:
# ❌ 타임아웃 (egress 차단)
docker exec app curl -I https://⟨evil.example⟩
# ✅ (allowlist)
docker exec app curl -I https://ghcr.io
```

👀 **보이는 것:** 허용된 레지스트리/저장소만 나가고 나머지는 막힘. AI든 앱이든 **구멍이 생겨도 데이터가 새지 않는다**.

💡 **느낄 것:** AI에게 서버를 맡기기 두려운 이유는 AI가 무엇을 가지고 나갈지 모르기 때문이다. 해법은 AI를 믿는 것이 아니라 **경계**를 두는 것이다. 시크릿은 손잡이(값이 아닌 참조)로만 건네고, 나가는 길은 화이트리스팅으로 막는다. 맡기되 안전하게 맡기는 방법이 이것이다. Day1에서 설정한 egress 통제가 여기서 AI 통제 장치가 된다.

---

🔀 **갈림길** — 다음엔 어디로?

- 전체를 꿰는 그림 — SADP 4통제지점 → [Part 7](../07-ai-governance.md)
- 배운 것을 우리 연구실에 적용하는 과제 만들기 → [AI로 내 교재 만들기: 우리 환경 과제](../ai/my-task.md)
- 다른 줄기를 골라 다시 → [🗺️ 지도](../map.md)
