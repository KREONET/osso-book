## 실습⑨′ 컨테이너 egress 가두기 — SADP outbound를 백엔드 컨테이너에

🎯 **목표:** 백엔드 앱 컨테이너가 **caddy·db 외엔 바깥으로 못 나가게**. "나갈 수 있는 곳"을 compose **네트워크 멤버십**으로 선언한다(방화벽 규칙 없이).

🖥️ **랩:** 작업 VM, caddy + app + db 스택(⑨ 연장). **app 이미지엔 `ping` 포함**(손놀림④로 구운 이미지).

⌨️ **(1) 가두기 전 — 컨테이너는 기본적으로 바깥이 열려 있다**
```bash
# 된다. (기본 bridge는 egress 허용)
docker compose exec app ping -c3 8.8.8.8
```

👀 100% 통함. 이게 **SADP가 짚는 구멍** — 털린 백엔드가 아무 데나 나간다(C2·데이터 유출).

⌨️ **(2) 네트워크 멤버십 = 허용 목록** (`compose.yaml`)
```yaml
services:
  caddy:
    image: caddy
    ports: ["443:443"]
    # edge=외부, web=app과 연결
    networks: [edge, web]
  app:
    # ping 포함해 구운 이미지(손놀림④)
    build: .
    # caddy(web) · db(data) 에만 붙음
    networks: [web, data]
  db:
    image: postgres
    # app 하고만
    networks: [data]
networks:
  # 외부 egress 가능 — caddy 전용
  edge: {}
  # 인터넷 차단
  web:  { internal: true }
  # 인터넷 차단
  data: { internal: true }
```
```bash
docker compose up -d
# 이제 100% loss (나갈 길이 없음)
docker compose exec app   ping -c3 8.8.8.8
# OK (web 공유)
docker compose exec app   ping -c2 caddy
# OK (data 공유)
docker compose exec app   ping -c2 db
# caddy는 여전히 OK (edge)
docker compose exec caddy ping -c2 8.8.8.8
```

👀 **보이는 것:** app의 `8.8.8.8`은 죽고 **caddy·db로는 멀쩡**. caddy는 외부도 app도 되고, db는 app하고만. **규칙 한 줄 없이** compose 멤버십만으로 egress를 그렸다.

💡 **느낄 것:** **app이 어느 네트워크에 붙었느냐 = 나갈 수 있는 곳 목록.** 이게 **SADP outbound 통제를 백엔드에** 적용한 실체 — 앱이 털려도 그 컨테이너는 바깥에 못 나간다.

💡 **같은 생각, 네 고도:** ⑥(호스트)→**여기(컨테이너)**→⑧(망 경계)→⑭(AI 에이전트). 전부 "나가는 길 기본 차단, 필요한 곳만 허용."

⚠️ `internal`은 "인터넷 차단"이지 "그 네트워크 안까지 차단"은 아니다 — 같은 net의 컨테이너끼린 통함. **목적지·포트 단위**로 더 조이려면 ③의 `DOCKER-USER`/`ufw-docker`(`ref/docker-ufw-nftables.html`)로 내려간다.

⚠️ `ping` 테스트엔 app 이미지에 `iputils-ping` 필요(없으면 `getent hosts caddy`/`curl -m3`로 대체) — **손놀림④에서 구울 때 심어둔다**.

---

🔀 **갈림길** — 다음엔 어디로?

- 같은 egress를 망 경계 고도에서 → [⑧ 망간접점·bastion](lab-08.md)
- 이 서비스를 https로 밖에 공개하려면 → [⑩ 진짜 인증서](lab-10.md)
- 네트워크 멤버십이 곧 허용목록인 이유 → [Part 4](../04-container.md)
- 길을 잃었다면 → [🗺️ 지도](../map.md)
