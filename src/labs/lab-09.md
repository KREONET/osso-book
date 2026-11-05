## 실습⑨ docker compose로 서비스 띄우고 → 써보고 → "도입할까?"를 ADR로

🎯 **목표:** 서비스를 **선언한 파일**로 띄우고, **실제로 써보고**, 도입 판단을 **ADR 한 장**으로 남긴다.

🖥️ **랩:** 작업 VM. 이미지 사전 pull 완료(전원만 켜면 됨).

⌨️ **(1) 띄우기** — 쓸만한 오픈소스 하나 (예: Uptime Kuma·Dozzle·NocoDB·Gitea·Outline 중 택1)
```bash
git clone ⟨https://github.com/kreonet/demo-⟨uptime-kuma⟩⟩ && cd demo-⟨uptime-kuma⟩
cat compose.yaml            # 뭘 띄우는지 먼저 읽는다
docker compose up -d
docker compose ps           # 뜬 컨테이너·포트
```

👀 `up -d` 몇 줄 만에 서비스가 뜸(이미지는 이미 받아둠). `ps`에 healthy·포트 매핑.

⌨️ **(2) 써보기** — 브라우저(lubuntu/noVNC)로 접속해 **진짜로 한 바퀴 돌려본다**
  예: Uptime Kuma면 모니터 1개 추가→알림 받기 / NocoDB면 표 하나 만들어 보기 / Gitea면 repo 하나 push.

👀 **보이는 것:** "설치형"이 아니라 **켜서 바로 쓰는** 경험. 맘에 안 들면 `docker compose down -v`로 흔적 없이 삭제.

⌨️ **(3) 판단을 남기기 — ADR 한 장** (`docs/adr/0001-⟨uptime-kuma⟩-도입.md`)
```markdown
# 1. ⟨Uptime Kuma⟩ 도입 여부
- 상태: 제안 | 날짜: 2026-xx-xx | 결정자: ⟨나⟩
## 맥락  무엇이 필요했나 (예: 사내 서비스 헬스 모니터링)
## 결정  채택/보류 + 한 줄 이유
## 대안  검토한 다른 것(예: Grafana+Prometheus = 과잉?) · 라이선스 확인
## 결과  유지보수·egress·백업(①②) 부담, 다시 볼 조건
```

👀 **보이는 것:** 결정이 머릿속이 아니라 **git에 남는 한 장**이 된다. 6개월 뒤 "왜 이거 썼지?"에 답이 있다.

💡 **느낄 것:** 설치 과정이 **파일 하나**(compose)에, 선택 이유가 **또 한 장**(ADR)에 적힌다. "서버에 뭘 깔았더라 + 왜 깔았더라"가 사라진다 — **섀도우IT 양지화의 실물**. 과잉설계(Grafana 풀스택)인지 적정(단일 컨테이너)인지도 ADR에서 걸러진다.

### ⑨ 보강 — 컨테이너 손에 익히기 (쓰기 → 들어가기 → 로컬처럼 → 고쳐쓰기 → 내가 굽기)

> ⑨ 하는 김에 익히는 **운영 손놀림**. 시간 되는 만큼 해보고, 깊은 건 gitbook.

⌨️ **① 안으로 들어가 보기** — 컨테이너는 격리됐지만 문은 있다
```bash
docker exec -it ⟨web⟩ bash        # 쉘 없으면 sh
  ps aux; cat /etc/os-release      # 프로세스가 사는 '집'을 구경
  exit
```

👀 호스트와 **다른 파일시스템·프로세스 공간**. "VM처럼 들어가 볼 수 있네." ⚠️ **아무나 `exec` 되면 격리가 무의미** — 누가 들어갈 수 있나는 권한 문제(⑭ AI에게 exec 줄지 신중).

⌨️ **② DB 컨테이너를 로컬 명령처럼 — `alias`**
```bash
# DB를 docker로 띄워두고, 그 안의 CLI를 alias 한 줄로 끌어낸다
alias influx='docker exec -it ⟨librenms_influx⟩ influx -database ⟨libre⟩'
alias psql='docker exec -it ⟨pg⟩ psql -U ⟨postgres⟩'
influx            # ← 이제 그냥 이렇게. 로컬에 설치한 것처럼.
# ~/.zshrc(또는 Day1 ① ohmyzsh 운영환경)에 넣어두면 장비마다 따라온다
```

💡 컨테이너를 "서비스"로만 보지 말고 **로컬 도구함**으로. `alias`가 그 다리 — ①의 ohmyzsh 운영환경과 **같은 결**(편의를 환경에 심는다).

⌨️ **③ 설정은 굽지 말고 — 추출→수정→마운트**
```bash
docker run --rm ⟨nginx⟩ cat /etc/nginx/nginx.conf > ./nginx.conf   # 기본값 꺼내기(1회성 run)
vi ./nginx.conf                                                    # 고치고
docker run -d -v ./nginx.conf:/etc/nginx/nginx.conf:ro ⟨nginx⟩     # 덮어서 실행
#   compose면 volumes: 에 한 줄
```

💡 ①에서 VM 이미지를 "**굽지 말고** 오프라인 수정"했듯, 컨테이너도 **설정은 이미지 밖에**. 이미지는 불변, 바뀌는 건 마운트로. 다시 띄워도 내 설정 그대로.

⌨️ **④ 내 앱을 이미지로 — 파이썬 앱 패키징**
```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 1000:1000                 # 비루트로
CMD ["python", "app.py"]
```
```bash
docker build -t ⟨myapp:1.0⟩ .          # 내 앱을 이미지로 '구움' = 어디서나 같게
docker run --rm -p 8000:8000 ⟨myapp:1.0⟩
```

👀 **보이는 것:** 내 코드 + 의존성이 **이미지 한 덩어리**로. 다른 서버에 `run` 한 줄이면 끝.

💡 **느낄 것:** ①이 "**남이** 구운 이미지를 받아 쓰기"였다면, 이건 "**내가** 굽기". `machine-id`/sysprep 없이도 "어디서나 같게"가 이미지의 본질. **⑭에서 AI가 만든 앱을 바로 이렇게 배포**한다. — 쓰기·들어가기·고쳐쓰기·굽기로 컨테이너가 손에 붙는다.

---

🔀 **갈림길** — 다음엔 어디로?

- 띄운 컨테이너가 아무 데나 나가지 못하게 → [⑨′ 컨테이너 egress](lab-09p.md)
- 도입 결정을 한 장으로 남기기 → [부록 C ADR](../C-adr.md)
- 이거 써도 되나 — 라이선스 → [부록 B](../B-licenses.md)
- 컨테이너 운영의 원리 → [Part 4](../04-container.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
