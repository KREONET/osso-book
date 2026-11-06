## 실습⑨ docker compose로 서비스 실행·사용하고 도입 여부를 ADR로 기록 {#lab-09}

🎯 **목표:** 서비스를 **선언한 파일**로 실행하고, 실제로 써 보고, 도입 판단을 **ADR 한 장**(Architecture Decision Record, 설계 결정과 그 이유를 적는 짧은 문서)으로 남긴다.

🖥️ **랩:** 작업 VM. 이미지 사전 pull 완료(전원만 켜면 됨).

> 컨테이너가 "격리된 프로세스"라는 말이 잘 와닿지 않는다면, 먼저 [Part 4 §4.0](../04-container.md)에서 fork·exec, `export`, `PATH`, `ps auxf`의 두 그루 나무를 직접 확인하고 오자. 5분이면 된다.

⌨️ **(1) 실행하기** — 쓸 만한 오픈소스 하나 (예: Uptime Kuma·Dozzle·NocoDB·Gitea·Outline 중 택1)
```bash
git clone ⟨https://github.com/kreonet/demo-⟨uptime-kuma⟩⟩ && cd demo-⟨uptime-kuma⟩
# 뭘 띄우는지 먼저 읽는다
cat compose.yaml
docker compose up -d
# 뜬 컨테이너·포트
docker compose ps
```

👀 `up -d` 몇 줄 만에 서비스가 실행됨(이미지는 이미 받아 둠). `ps`에 healthy·포트 매핑.

⌨️ **(2) 써보기** — 브라우저(lubuntu/noVNC)로 접속해 **실제로 한 번 사용해 본다**
  예: Uptime Kuma면 모니터 1개 추가→알림 받기 / NocoDB면 표 하나 만들어 보기 / Gitea면 repo 하나 push.

👀 **보이는 것:** 설치 과정 없이 **켜서 바로 쓰는** 경험. 마음에 들지 않으면 `docker compose down -v`로 흔적 없이 삭제.

⌨️ **(3) 판단을 남기기 — ADR 한 장** (`docs/adr/0001-⟨uptime-kuma⟩-도입.md`)
```markdown
# 1. ⟨Uptime Kuma⟩ 도입 여부
- 상태: 제안 | 날짜: 2026-xx-xx | 결정자: ⟨나⟩
## 맥락  무엇이 필요했나 (예: 사내 서비스 헬스 모니터링)
## 결정  채택/보류 + 한 줄 이유
## 대안  검토한 다른 것(예: Grafana+Prometheus = 과잉?) · 라이선스 확인
## 결과  유지보수·egress·백업(①②) 부담, 다시 볼 조건
```

👀 **보이는 것:** 결정이 머릿속에만 있지 않고 **git에 남는 한 장**이 된다. 6개월 뒤 왜 이것을 골랐는지 물을 때 답이 남아 있다.

💡 **느낄 것:** 설치 과정은 **파일 하나**(compose)에, 선택 이유는 **또 한 장**(ADR)에 적힌다. 서버에 무엇을 왜 설치했는지 잊어버리는 일이 없어진다. 섀도우 IT(조직이 모르는 채 운영되는 비공식 시스템)를 공식 관리 안으로 끌어내는 실제 방법이다. 과잉 설계(Grafana 풀스택)인지 적정 수준(단일 컨테이너)인지도 ADR을 쓰면서 걸러진다.

### ⑨ 보강 — 컨테이너 기본 조작 (접속, alias, 설정 수정, 이미지 빌드) {#lab-09-more}

> ⑨를 하면서 함께 익히는 **운영 손놀림**이다. 시간이 되는 만큼 해 보고, 깊은 내용은 gitbook에서 본다.

⌨️ **① 안으로 들어가 보기** — 컨테이너는 격리됐지만 문은 있다
```bash
# 쉘 없으면 sh
docker exec -it ⟨web⟩ bash
  # 프로세스가 사는 '집'을 구경
  ps aux; cat /etc/os-release
  exit
```

👀 호스트와 **다른 파일시스템·프로세스 공간**. VM처럼 안에 들어가 볼 수 있다. ⚠️ **아무나 `exec`할 수 있으면 격리는 의미가 없다.** 누가 들어갈 수 있는지는 권한 문제다(⑭에서 AI에게 exec 권한을 줄지는 신중히 판단한다).

⌨️ **② DB 컨테이너를 로컬 명령처럼 — `alias`**
```bash
# 예측: 호스트에 psql이 있을까?
which psql
# PostgreSQL을 컨테이너로 띄운다 — 이름은 pg, 포트는 밖에 열지 않는다
docker run -d --name pg -e POSTGRES_HOST_AUTH_METHOD=trust postgres:16
# 그 안의 psql을 alias 한 줄로 끌어낸다
alias psql='docker exec -it pg psql -U postgres'
# 이제 로컬에 설치한 것처럼
psql -c 'select version();'
# 정체 확인
type psql
```

출력 예:
```text
psql not found
…
 PostgreSQL 16.x (Debian 16.x-1.pgdg120+1) on x86_64-pc-linux-gnu, …
psql is an alias for docker exec -it pg psql -U postgres
```

> 현장에서는 이렇게 쓴다. LibreNMS가 InfluxDB를 컨테이너로 실행해 두었다면:
>
> `alias influx='docker exec -it librenms_influxdb influx -database librenms'`
>
> 컨테이너 이름은 `docker ps`의 NAMES 열에서, DB 이름은 그 서비스의 설정에서 확인한다. alias는 `~/.zshrc`(①의 ohmyzsh 운영환경)에 넣어 두면 장비마다 따라온다.

👀 **보이는 것:** 호스트에는 `psql`이 없는데 `psql`이 실행된다. `type`으로 보면 정체는 `docker exec`. 버전은 컨테이너 이미지의 것(16)이다.

> 💭 `docker inspect pg --format '{{json .Config.Env}}'`도 실행해 보자. `PATH` 끝에 `/usr/lib/postgresql/16/bin`이 붙어 있다. 컨테이너 안에서 `psql`을 경로 없이 부를 수 있는 이유가 이것이다([Part 4 §4.0](../04-container.md)).

💡 컨테이너는 서비스뿐 아니라 **로컬 도구함**으로도 쓸 수 있다. `alias`가 그 연결 고리이고, ①의 ohmyzsh 운영환경과 같은 생각이다(편의를 환경에 넣어 둔다). 클라이언트를 따로 설치하지 않으니 **서버와 버전이 어긋날 일도 없다.** 다 봤으면 `docker rm -f pg`로 지운다.

⚠️ `POSTGRES_HOST_AUTH_METHOD=trust`는 암호 없이 접속하게 하는 **실습 전용** 설정이다. 포트를 밖에 열지 않았기 때문에 이 VM 안에서만 쓸 수 있다. 실제 서비스에서는 쓰지 않는다.

⌨️ **③ 설정은 굽지 말고 — 추출→수정→마운트**
```bash
# 기본값 꺼내기(1회성 run)
docker run --rm ⟨nginx⟩ cat /etc/nginx/nginx.conf > ./nginx.conf
# 고치고
vi ./nginx.conf
# 덮어서 실행
docker run -d -v ./nginx.conf:/etc/nginx/nginx.conf:ro ⟨nginx⟩
#   compose면 volumes: 에 한 줄
```

💡 ①에서 VM 이미지를 다시 굽지(bake, 설정을 넣어 이미지를 새로 빌드하는 일) 않고 오프라인으로 수정했듯이, 컨테이너도 **설정은 이미지 밖에** 둔다. 이미지는 그대로 두고, 바뀌는 것은 마운트로 넣는다. 다시 실행해도 내 설정이 그대로 적용된다.

⌨️ **④ 내 앱을 이미지로 — 파이썬 앱 패키징**
```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
# root가 아닌 일반 사용자(UID 1000)로 실행 — non-root
USER 1000:1000
CMD ["python", "app.py"]
```
```bash
# 내 앱을 이미지로 '구움' = 어디서나 같게
docker build -t ⟨myapp:1.0⟩ .
docker run --rm -p 8000:8000 ⟨myapp:1.0⟩
```

👀 **보이는 것:** 내 코드와 의존성이 **이미지 한 덩어리**가 된다. 다른 서버에서도 `run` 한 줄이면 실행된다.

💡 **느낄 것:** ①이 남이 구운 이미지를 받아 쓰는 일이었다면, 이것은 **내가** 굽는 일이다. `machine-id`/sysprep 없이도 어디서나 같게 실행되는 것이 이미지의 본래 목적이다. ⑭에서는 AI가 만든 앱을 이 방식으로 배포한다. 쓰기·들어가기·고쳐 쓰기·굽기를 거치며 컨테이너 다루는 법이 손에 익는다.

---

🔀 **갈림길** — 다음엔 어디로?

- 실행한 컨테이너가 아무 데나 나가지 못하게 → [⑨′ 컨테이너 egress](lab-09p.md)
- 도입 결정을 한 장으로 남기기 → [부록 C ADR](../C-adr.md)
- 이것을 써도 되나 — 라이선스 → [부록 B](../B-licenses.md)
- 컨테이너 운영의 원리 → [Part 4](../04-container.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
