# 부록 F. 용어집

알파벳 순(숫자 먼저)입니다.

| 용어 | 설명 |
|---|---|
| 802.1X | 포트 단위 인증 표준. 인증된 단말만 네트워크에 붙게 한다(NAC의 구현 수단). |
| ACME | 인증서를 자동 발급·갱신하는 프로토콜(RFC 8555). Let's Encrypt가 사용. |
| ADR | Architecture Decision Record. 결정과 이유를 짧게 남기는 문서. → [부록 C](C-adr.md) |
| bastion | 내부망 진입 전용 관문 서버. 외부 접속을 여기 한 곳으로 모은다. |
| CA | 인증기관. 서버·클라이언트 인증서에 서명해 신뢰를 보증한다. |
| CCE | Common Configuration Enumeration. 보안 설정 항목의 표준 식별자. |
| CVE | 공개된 취약점의 표준 식별 번호(예: CVE-2024-xxxx). |
| DNAT | 목적지 주소 변환. 외부 포트를 내부 서비스로 포워딩할 때 쓴다. |
| DNS-01 | ACME 검증 방식. DNS TXT 레코드로 소유를 증명한다(와일드카드 가능). |
| DOCKER-USER | Docker가 FORWARD 앞에 두는 사용자 정의 iptables 체인. 컨테이너 방화벽을 여기서 건다. |
| egress | 나가는 트래픽. "어느 고도에서 egress를 통제하나"가 이 책의 한 줄. |
| forward-auth | 리버스 프록시가 요청을 인증 서버에 먼저 물어보고 통과시키는 방식. |
| HNUM | ohmyzsh 프롬프트에 표시되는 셸 히스토리 번호. `!N`으로 그 번호의 명령을 다시 실행한다. |
| HTTP-01 | ACME 검증 방식. 80포트의 지정 경로로 소유를 증명한다. |
| INPUT / FORWARD | iptables/nftables 체인. INPUT은 호스트 자신에게 오는 것, FORWARD는 호스트를 거쳐 가는 것(컨테이너 트래픽). |
| internal network | 외부로 나가는 경로가 없는 Docker 네트워크(`internal: true`). |
| IPMI | 서버 보드(BMC)의 원격 관리 인터페이스. 전원·콘솔 제어. 외부에 노출하지 말 것. |
| LUKS | 리눅스 디스크 암호화 표준. 분실·도난 시 데이터 보호. |
| machine-id | `/etc/machine-id`. 머신 고유 ID로, 이미지 복제 시 반드시 리셋해야 한다. |
| mTLS | 서버뿐 아니라 클라이언트도 인증서로 인증하는 상호 TLS. |
| NAC | Network Access Control. 단말 인증·상태 검사 후 네트워크 접근 허용. |
| non-root (비루트) | 프로세스를 root가 아닌 일반 사용자로 실행하는 것. 뚫려도 피해가 그 사용자 권한에 머문다. Dockerfile의 `USER`. → [Part 4 §4.5](04-container.md) |
| OIDC | OpenID Connect. OAuth 2.0 위의 로그인(인증) 표준. |
| ohmyzsh 태그 (TAG1/TAG2) | 물리 서버의 `/etc/tag1`(모델)·`/etc/tag2`(자산 ID)를 프롬프트에 표시하는 강사 개인 컨벤션. 소개만 하고 실습하지 않는다. |
| 호스트명 규칙 | VM 구분용 이름 짓기 예: `⟨IP 끝자리 3자리⟩-⟨OS약어⟩-⟨용도⟩` (예: `042-u24-web`). |
| ProxyJump | `ssh -J`. 중간 서버(bastion)를 경유해 대상에 접속한다. |
| PUEL | VirtualBox Extension Pack의 라이선스. 개인·평가용만 무료, 기관은 유료. → [부록 B](B-licenses.md) |
| SNAT | 출발지 주소 변환. 내부 대역이 외부로 나갈 때(masquerade) 쓴다. |
| SSPL | MongoDB 등이 쓴 Server Side Public License. 서비스 제공 시 소스 공개 의무, OSI 비승인. |
| sysprep | 이미지 복제 전 머신 정체성(machine-id·호스트키·로그)을 지우는 작업. |
| V2P | Virtual to Physical. 가상 이미지를 물리 서버로 복원·이식하는 것. |

관련: [부록 D 치트시트](D-cheatsheet.md)
