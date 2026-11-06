## 실습⑬ AI에게 서버 조사 맡기기 (읽기 전용) {#lab-13}

🎯 **목표:** 범위를 좁힌 SSH를 AI에게 주고 **읽기 전용** 점검을 시킨다.

🖥️ **랩:** 작업 VM에서 Claude Code(또는 ⟨chat.내부LLM⟩) + bastion 경유 격리 VM들.

⌨️ **해보기**
```bash
# AI에게 줄 ssh 키는 읽기 전용 조사 계정(ops-ro). config는 ⑧에서 만든 ProxyJump 그대로.
# Claude Code 실행 후, 예:
claude
#  "ssh로 isolated1~3 돌면서 OS버전·열린포트·docker ps·디스크사용량 표로 정리해줘.
#   변경은 하지 말고 조사만."
```

👀 **보이는 것:** AI가 여러 호스트를 ssh로 돌며 `ss -tlnp`, `docker ps`, `df -h`를 모아 **표**로 요약. bastion을 자동으로 경유.

💡 **느낄 것:** 서버 20대의 상태 파악이 사람에게는 반나절, AI에게는 몇 분이다. 중요한 점은 AI가 **준 권한 안에서만** 움직인다는 것이다. 읽기 계정을 주었으니 조사만 한다. 권한 설계가 곧 안전장치다. 이것이 [위임 사다리](../07-ai-governance.md#7-1-delegation-ladder)의 1단계(조사)다. 그다음 칸으로 올라가는 조건은 Part 7에서 다룬다.

---

🔀 **갈림길** — 다음엔 어디로?

- AI가 만든 앱을 가둬서 배포하기 → [⑭ AI 배포 캡스톤](lab-14.md)
- AI에게 얼마나 맡길까 — 권한 경계 → [Part 7](../07-ai-governance.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
