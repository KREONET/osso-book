# Part 4. 컨테이너 운영

> 담당 랩 [실습⑨](labs/lab-09.md) · [실습⑨′](labs/lab-09p.md) + 손놀림.

이 장의 한 줄: **컨테이너는 "작은 VM"이 아니라 "선언된 프로세스"다.** 선언은 파일에 남고, 나갈 수 있는 길도 선언으로 그린다.

## 4.1 컨테이너 vs VM — 과잉설계 경계

**멘탈모델.** VM은 *컴퓨터 한 대*를 흉내 낸다(커널까지 따로). 컨테이너는 *호스트 커널 위의 격리된 프로세스*다(namespace + cgroup). 그래서 VM은 "OS를 운영"하고, 컨테이너는 "프로그램 하나를 운영"한다.

| | VM | 컨테이너 |
|---|---|---|
| 격리 단위 | 커널 포함 한 대 | 프로세스(호스트 커널 공유) |
| 기동 | 분 단위 | 초 단위 |
| 격리 강도 | 강함(하이퍼바이저 경계) | 상대적으로 약함(커널 공유) |
| 산출물 | 디스크 이미지(수 GB) | 레이어 이미지(수십~수백 MB) |
| 잘 맞는 일 | 다른 OS, 강한 멀티테넌트 | 서비스 하나, 재현 가능한 배포 |

**왜 이게 중요한가.** 커널을 공유한다는 건 *컨테이너 탈출 = 호스트 장악*이라는 뜻이다. 그래서 신뢰할 수 없는 코드를 돌릴 땐 VM 경계가 여전히 낫고, 컨테이너에는 비루트·egress 차단 같은 보강이 붙는다(4.4, 4.5).

**과잉설계 경계.** 컨테이너가 쉬워지면 반대 실수가 온다. "서비스 헬스 보고 싶다"에 Prometheus + Grafana + Alertmanager + exporter를 짓는 것.

- 필요가 *"죽었는지 알림"* 이면 → 단일 컨테이너(예: Uptime Kuma)로 끝.
- 필요가 *"수백 개 지표의 시계열 분석"* 일 때 비로소 풀스택.
- 판단 기준: **내가 6개월 뒤에도 혼자 고칠 수 있는가.** 컨테이너 수 = 유지보수 부담 수.

이 판단은 머리에만 두지 말고 ADR 한 장으로 남긴다([실습⑨](labs/lab-09.md)의 "대안" 칸). 과잉설계는 대개 *대안을 적지 않아서* 생긴다.

**확인.**

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'   # 내가 몇 개를 돌리고 있나
docker stats --no-stream                                          # 그 값어치를 하나
uname -r; docker exec <ctr> uname -r                              # 같은 커널 = 공유의 증거
```

**바깥 링크.** [Docker: What is a container](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/) · [Linux namespaces(man7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)

## 4.2 선언형 — compose, "뭘 깔았더라"가 사라진다

**멘탈모델.** 명령형은 *"무엇을 했는가"* 의 기록(셸 히스토리, 기억), 선언형은 *"무엇이어야 하는가"* 의 기록(파일)이다. 서버 상태 = `compose.yaml` + 데이터 볼륨. 그 외는 전부 버려도 되는 것이다.

```yaml
# compose.yaml — 이 한 파일이 "설치 문서"를 대신한다
services:
  kuma:
    image: louislam/uptime-kuma:1      # 버전을 고정(재현성)
    restart: unless-stopped
    ports: ["127.0.0.1:3001:3001"]     # 바깥에 열지 않고 로컬만
    volumes:
      - kuma-data:/app/data            # 지켜야 할 건 이것뿐
volumes:
  kuma-data: {}
```

**왜.**

- **재현성:** 서버가 망가져도 지우고 `up`하면 복구된다. "복구 절차"가 곧 "설치 절차"라 따로 문서화할 게 없다.
- **감사 가능:** 파일이 git에 있으면 *누가 언제 왜* 바꿨는지가 남는다. 섀도우 IT가 양지로 나오는 지점이다.
- **데이터와 분리:** 컨테이너는 소모품, 볼륨만 자산이다. 백업 대상이 명확해진다([Part 1](01-image.md)·백업 장 참고).

**확인.** 복구 리허설을 한 번은 해본다.

```bash
docker compose up -d && docker compose ps
docker compose down                 # 컨테이너·네트워크 삭제 (볼륨은 유지)
docker compose up -d                # 그대로 복구 — 데이터도 살아 있다
docker compose down -v              # -v 는 볼륨까지 삭제: 되돌릴 수 없음
docker compose config               # 변수 치환이 끝난 최종 선언을 보기
```

> 함정: `latest` 태그와 이름 없는 볼륨은 선언형을 조용히 망친다. 버전은 고정하고, 지킬 데이터는 이름 있는 볼륨이나 명시적 경로로 둔다.

**바깥 링크.** [Compose specification](https://docs.docker.com/reference/compose-file/) · [Docker: Volumes](https://docs.docker.com/engine/storage/volumes/)

## 4.3 손에 익히기 — exec·alias·설정 추출/마운트·패키징

선언형이어도 손은 쓴다. 네 가지 손놀림이 컨테이너를 "남의 서비스"에서 "내 도구"로 바꾼다. 따라 하기는 [실습⑨ 보강](labs/lab-09.md)에 있고, 여기선 *왜 그렇게 하는지*만 짚는다.

### (1) `docker exec -it` — 컨테이너 안으로

```bash
docker compose exec app sh          # compose 서비스 이름으로
docker exec -it <ctr> bash          # 컨테이너 이름으로 (쉘이 없으면 sh)
```

- `exec`는 *이미 도는* 컨테이너의 네임스페이스에 새 프로세스를 끼워 넣는다. `run`은 *새* 컨테이너를 만든다. 혼동하면 "고쳤는데 사라졌다"가 된다.
- 안에서 고친 파일은 컨테이너와 함께 사라진다. **구경과 디버깅에만** 쓰고 수정은 4.3 (3)으로.
- 보안: `docker` 그룹 = 사실상 호스트 root. `exec` 권한은 곧 그 컨테이너의 권한이다([Part 5](05-trust.md)·AI 에이전트에 줄 때 특히 유의).

### (2) `alias` — 컨테이너를 로컬 도구함으로

```bash
alias psql='docker compose exec -T db psql -U postgres'
alias redis-cli='docker exec -it redis redis-cli'
psql -c 'select now();'             # 로컬에 설치한 것처럼
```

- 호스트에 DB 클라이언트를 *설치하지 않는다.* 버전도 서버와 같은 이미지에서 온다 — 클라이언트/서버 버전 불일치가 사라진다.
- 파이프·리다이렉트를 쓸 땐 `-T`(또는 `-i`만), 대화형이면 `-it`. TTY 유무가 가장 흔한 실패 원인이다.
- `~/.zshrc`에 넣어 두면 장비마다 따라온다.

### (3) 설정: 추출 → 수정 → 볼륨 마운트 ("굽지 말고 마운트")

```bash
docker run --rm nginx cat /etc/nginx/nginx.conf > ./nginx.conf   # 1) 기본값 추출
vi ./nginx.conf                                                  # 2) 수정
```

```yaml
services:
  web:
    image: nginx:1.27
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro                    # 3) 덮어 마운트
```

- 이미지는 **불변**, 바뀌는 것은 **밖에서 주입**. 이미지를 업그레이드해도 내 설정은 그대로다.
- 설정을 이미지에 `COPY`로 굽는 건 *설정이 바뀔 때마다 빌드*가 필요해 느리고, 비밀(키·비밀번호)이 레이어에 남을 위험도 있다.
- `:ro`로 마운트하면 컨테이너가 털려도 설정을 못 바꾼다.

### (4) Dockerfile — 내 앱 패키징

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 1000:1000
CMD ["python", "app.py"]
```

- "내 PC에서는 되는데"가 이미지 한 덩어리로 해결된다. 다른 서버에선 `docker run` 한 줄이다.
- 의존성 파일을 먼저 `COPY`하는 순서는 *캐시* 때문이다(4.5).

**확인.**

```bash
docker compose exec app id          # 지금 누구로 도나 (root면 4.5로)
docker build -t myapp:1.0 . && docker run --rm -p 8000:8000 myapp:1.0
```

**바깥 링크.** [docker exec](https://docs.docker.com/reference/cli/docker/container/exec/) · [Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)

## 4.4 네트워크 멤버십 = egress 허용 목록

**멘탈모델.** 컨테이너의 "나갈 수 있는 곳"은 방화벽 규칙이 아니라 **어느 네트워크에 붙어 있느냐**로 정해진다. compose의 `networks:` 목록이 곧 *허용 목록*이다. 이것이 SADP의 outbound 통제를 백엔드에 적용한 모습이다([실습⑨′](labs/lab-09p.md)).

```text
              인터넷
                 |
   [edge]  ──── caddy ──── [web, internal] ──── app ──── [data, internal] ──── db
 (egress 가능)                (인터넷 차단)                  (인터넷 차단)
```

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["443:443"]
    networks: [edge, web]       # 바깥과 app 사이의 문지기
  app:
    build: .
    networks: [web, data]       # 붙은 곳 = 말할 수 있는 곳 (edge 없음)
  db:
    image: postgres:16
    networks: [data]
networks:
  edge: {}                      # 기본 bridge: NAT로 외부 나감
  web:  { internal: true }
  data: { internal: true }
```

**왜.** 백엔드가 털렸을 때의 피해는 대부분 *나가는 길*에서 결정된다. C2 통신, 데이터 유출, 추가 도구 다운로드가 전부 outbound다. 앱이 인터넷에 갈 일이 없다면 길 자체를 없앤다. 방화벽 규칙을 쓰지 않아도 되고, 선언이 곧 정책이라 *리뷰 가능*하다.

`internal: true`는 Docker가 그 네트워크에 외부 라우팅·NAT(masquerade)를 만들지 않는다는 뜻이다. 게이트웨이가 없으니 `8.8.8.8`로 갈 길이 없다.

**확인.** 가두기 전/후를 반드시 대조한다.

```bash
docker compose exec app ping -c3 8.8.8.8    # internal 적용 후: 100% loss
docker compose exec app ping -c2 caddy      # OK (web 공유)
docker compose exec app ping -c2 db         # OK (data 공유)
docker compose exec caddy ping -c2 8.8.8.8  # OK (edge)
docker network inspect <project>_web --format '{{.Internal}}'   # true
```

> `ping`이 없으면 `getent hosts caddy`, `curl -m3 https://example.com`로 대체한다.

**한계 — 이것이 전부는 아니다.**

- `internal`은 "인터넷 차단"이지 "그 네트워크 *안*까지 차단"이 아니다. 같은 net의 컨테이너끼리는 서로 모든 포트로 통한다. 예: app이 털리면 같은 `data` net의 db 전 포트가 열려 있다.
- 서비스를 `edge`에 붙이는 순간 그 컨테이너는 *전면 egress 허용*이다. 붙이는 곳을 최소화하고, caddy 같은 문지기만 붙인다.
- 목적지·포트 단위로 조이려면 호스트의 `DOCKER-USER` 체인으로 내려간다 → [Part 2](02-firewall.md). 컨테이너 트래픽이 `INPUT`이 아니라 `FORWARD`를 지나므로 일반 방화벽 규칙이 안 먹는 이유도 거기서 다룬다.

**같은 생각, 여러 고도.** 호스트(실습⑥) → **컨테이너(여기)** → 망 경계(실습⑧) → AI 에이전트(실습⑭). 전부 "나가는 길은 기본 차단, 필요한 곳만 허용"이다.

**바깥 링크.** [Docker: Networking](https://docs.docker.com/engine/network/) · [Compose: networks (`internal`)](https://docs.docker.com/reference/compose-file/networks/#internal) · [Packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

## 4.5 이미지 빌드 — Dockerfile, 비루트, 멀티스테이지, 대역 겹침

### 비루트 (`USER`)

**멘탈모델.** 컨테이너 안의 root는 호스트 root와 *같은 UID 0*이다. 격리가 뚫리거나 볼륨을 마운트한 상태라면 그 힘이 그대로 호스트에 닿는다. 앱은 root일 이유가 없다.

```dockerfile
RUN useradd --uid 10001 --no-create-home app
USER 10001:10001
```

- 숫자 UID로 지정하면 오케스트레이터가 "비루트인지" 검증하기 쉽다.
- 1024 미만 포트는 못 연다 → 8080처럼 높은 포트에서 받고 앞단(caddy)이 443을 맡는다.
- 마운트한 볼륨의 소유권이 UID와 맞아야 쓰기가 된다. "Permission denied"의 대부분이 이것이다.

```bash
docker compose exec app id                                  # uid=10001 이어야 함
docker inspect -f '{{.Config.User}}' myapp:1.0              # 비면 root
```

### 멀티스테이지

**멘탈모델.** *빌드 도구는 빌드에만, 실행에는 결과물만.* 컴파일러·패키지 매니저·소스는 공격 표면이자 용량이다. 최종 이미지에 넣지 않는다.

```dockerfile
# --- 빌드 단계: 도구 잔뜩 ---
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download                 # 의존성 먼저 → 캐시 적중
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

# --- 실행 단계: 결과물만 ---
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
USER 65532:65532
ENTRYPOINT ["/app"]
```

- 결과 이미지에는 쉘도 `ping`도 없다. 디버깅이 불편한 대신 털려도 쓸 도구가 없다. 트레이드오프를 알고 고른다(실습⑨′은 `ping` 확인용으로 일부러 넣은 것).
- 레이어는 순서가 캐시다. *자주 바뀌는 것을 아래에* 둔다. 일반론은 아래 표준 문서로.

### 컨테이너 대역이 기관망·VPN과 겹치지 않게 — `default-address-pools`

**멘탈모델.** Docker는 네트워크를 만들 때마다 기본 풀(`172.17.0.0/16`부터 `172.31.x`, 이어서 `192.168.x`)에서 대역을 *자동* 할당한다. 기관 내부망이나 VPN이 같은 대역을 쓰면, 호스트는 그쪽 목적지를 *컨테이너 브리지로 보내버린다.* 증상은 "컨테이너를 띄우자 VPN/사내 서버 접속이 끊긴다", 또는 반대로 컨테이너가 사내 서버에 못 간다.

원인은 **라우팅 테이블에서 더 구체적인(또는 나중에 생긴) 경로가 이긴다**는 것뿐이다. 컨테이너 대역과 겹치는 목적지는 방화벽 이전에 라우팅 단계에서 이미 갈 곳을 잃는다.

**확인.** 먼저 지금 무엇이 겹치는지 본다.

```bash
ip route                                             # 사내망/VPN 경로와 docker 브리지 경로 비교
docker network ls -q | xargs docker network inspect \
  --format '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

**처방.** 기관이 쓰지 않는 대역을 *미리 한 번* 정해 두고, 풀을 그 안으로 제한한다.

```json
// /etc/docker/daemon.json
{
  "bip": "172.31.255.1/24",
  "default-address-pools": [
    { "base": "172.31.0.0/16", "size": 24 }
  ]
}
```

- `bip`: 기본 브리지 `docker0`의 대역. `default-address-pools`: compose 등이 만드는 *사용자 정의* 네트워크의 풀(`base`를 `size` 크기로 잘라 쓴다).
- 위 예(`172.31.0.0/16`, `/24`씩)는 네트워크 약 255개분이다. 프로젝트가 많으면 `size`를 줄이거나 base를 넓힌다.
- **적용:** `sudo systemctl restart docker`. 이미 만든 네트워크는 그대로이므로 `docker compose down && up -d`로 재생성한다. 재시작은 컨테이너를 멈추니 점검 창에서 한다.
- 대역은 **기관 IP 담당과 합의한 값**을 쓴다. 이 예시는 문서용이다.
- firewalld를 쓰는 서버라면 브리지 인터페이스가 어느 zone에 속하는지도 함께 확인한다. Docker 트래픽이 firewalld와 *다른 길*로 흐르는 이유는 [Part 2](02-firewall.md)에서 다룬다.

**바깥 링크.** [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/) · [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/) · [Daemon configuration (`default-address-pools`)](https://docs.docker.com/engine/daemon/) · [Docker and firewalld](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-firewalld)
