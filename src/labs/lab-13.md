## 실습⑬ AI에게 서버를 조사시키기 (읽기만)

🎯 **목표:** 범위를 좁힌 SSH를 AI에게 주고 **읽기 전용** 점검을 시킨다.

🖥️ **랩:** 작업 VM에서 Claude Code(or ⟨chat.내부LLM⟩) + bastion 경유 격리 VM들.

⌨️ **해보기**
```bash
# AI에게 줄 ssh 키는 읽기 위주 계정. config는 ⑧에서 만든 ProxyJump 그대로.
claude      # Claude Code 실행 후, 예:
#  "ssh로 isolated1~3 돌면서 OS버전·열린포트·docker ps·디스크사용량 표로 정리해줘.
#   변경은 하지 말고 조사만."
```

👀 **보이는 것:** AI가 여러 호스트를 ssh로 돌며 `ss -tlnp`, `docker ps`, `df -h`를 모아 **표**로 요약. bastion을 알아서 경유.

💡 **느낄 것:** "서버 20대 상태 파악"이 사람 반나절→AI 몇 분. 핵심은 **준 권한 안에서만** 움직인다는 것 — 읽기 계정을 줬으니 조사만 한다. 권한 설계가 곧 안전장치.

