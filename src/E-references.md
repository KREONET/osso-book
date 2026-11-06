# 부록 E. 참고자료 {#appendix-e}

## 이 책으로 옮겨 온 문서 {#e-migrated}

아래 문서의 내용은 모두 이 책 안에 옮겨 두었습니다. 원래 문서를 찾지 않아도 오른쪽 절만 읽으면 됩니다.

* 「서버 납품 가이드」 → [부록 K](K-delivery.md)
* 「Multi-Homed 설정」 → [Part 3 §3.2](03-network.md)
* 「MTU 9000 Ping」 → [Part 3 §3.6](03-network.md)
* 「회사 보안 장비 및 소프트웨어」 → [Part 5 §5.5](05-trust.md), [Part 6 §6.5](06-access.md) (개념만)
* 「2023 인턴 읽을거리」 → [§1.2](01-image.md), [§1.6](01-image.md), [§1.8](01-image.md), [부록 D §7](D-cheatsheet.md), 이 부록의 「혼자 더 공부할 때」
* 「PC 공통 환경 설정 (2025)」 → [부록 J](J-workstation.md), [부록 G](G-packages.md), [Part 1 §1.3 점 파일](01-image.md)

## 분석 노트 {#e-notes}

* "Docker와 ufw/nftables" — Docker가 `DOCKER-USER`·nat 체인을 통해 ufw를 우회하는 원리와 대응
* "Docker와 firewalld(zone vs policy)" — firewalld의 zone/policy 모델과 Docker 트래픽의 관계

## 학습 설계 {#e-learning-design}

* 김창준, 『야생 학습 — 불확실성의 시대, 학습은 왜 더 중요해졌는가』, 2026년 11월 2일. <https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=404995378>
  — 이 책의 [지도](map.md)·갈림길 구성(지도·옵션·가설·액관해)과 [AI로 내 교재 만들기](ai/index.md)의 원칙이 여기서 왔습니다.

## 이 책이 기대는 개념의 원전 {#e-sources}

* 장민석, "AI 시대, 빠르고 안전한 앱 프로토타입 — 섀도우 IT를 양지로 올리는 법", 출연연 오픈소스 테크데이 2026, 2026-10-07. <https://msjang.github.io/ostday26-sadp/> (저장소 <https://github.com/msjang/ostday26-sadp>, CC BY-NC-ND 4.0)
  — **SADP**(Secure Application Deployment Profile) 4통제지점과 정책 제안 아홉 가지의 원문. [Part 7 §7.5](07-ai-governance.md).
* Trusted CI, "The Trusted CI Framework". <https://www.trustedci.org/framework/core> — 요건만 정하고 구현은 묻지 않는 보안 프레임워크. SADP가 "프로파일"을 택한 근거.

## 더 공부하기 — 기술 서적과 강의 구독 {#e-further-study}

* **O'Reilly Learning** <https://learning.oreilly.com> — 표지에 동물·곤충·식물 그림이 있는 그 출판사의 책·동영상 강의·실습 환경을 묶은 구독 서비스입니다(옛 이름 Safari Books Online). 개인 구독은 비싼 편이니, 소속 기관이나 학회 회원 혜택으로 쓸 수 있는지 먼저 확인합니다.
* **ACM 학생 회원 e-러닝** — ACM 학생 회원(연회비가 낮은 편)에게 온라인 강의·전자책 묶음이 제공됩니다. 제휴처는 바뀝니다. 한때 O'Reilly가 포함됐다가 빠졌고, 2023년 기준으로는 Skillsoft(Percipio) 강의·전자책이 제공됐습니다. 가입 전 현재 혜택을 확인하세요.
* **읽는 요령:** 영어로 된 기술서도 스택오버플로 답변을 읽을 정도면 충분히 읽을 수 있습니다. 처음부터 끝까지 읽기보다, **책 본문을 검색해 필요한 부분만 발췌해 읽는** 방식이 실무에서 오래 쓸모가 있습니다. 검색 결과로 나온 단편 답변과 달리, 책은 그 답의 앞뒤 맥락(왜 그런가)을 함께 줍니다.

## 연구·통계 {#e-research}

* Asim Kadav, Michael M. Swift, "Understanding Modern Device Drivers", ASPLOS '12, 2012. <https://pages.cs.wisc.edu/~kadav/app/papers/driverstudy.pdf>
  — Linux 2.6.37.6의 드라이버 3,217개 분석. "드라이버가 리눅스 코드의 70%", 네트워크 드라이버는 드라이버 코드의 25%. [Part 4 §4.0](04-container.md)의 커널 코드 구성표와 비교.
* heise online, "Linux breaks through the 40 million line mark", 2025-01-25. <https://www.heise.de/en/news/Linux-breaks-through-the-40-million-line-mark-10256571.html>

## 외부 표준·공식 문서 {#e-standards}

| 주제 | 링크 |
|---|---|
| mdBook | <https://rust-lang.github.io/mdBook/> |
| OpenSSH 매뉴얼 | <https://www.openssh.com/manual.html> |
| Docker 문서 | <https://docs.docker.com/> |
| Docker와 방화벽 | <https://docs.docker.com/engine/network/packet-filtering-firewalls/> |
| RFC 8555 (ACME) | <https://www.rfc-editor.org/rfc/rfc8555> |
| RFC 5280 (X.509) | <https://www.rfc-editor.org/rfc/rfc5280> |
| RFC 5737 (문서용 IPv4 대역 — 이 책의 예시 주소) | <https://www.rfc-editor.org/rfc/rfc5737> |
| OpenID Connect Core | <https://openid.net/specs/openid-connect-core-1_0.html> |
| nftables wiki | <https://wiki.nftables.org/> |
| firewalld | <https://firewalld.org/documentation/> |
| libguestfs | <https://libguestfs.org/> |
| OSI 승인 라이선스 | <https://opensource.org/licenses> |
| MADR | <https://adr.github.io/madr/> |

관련: [부록 A 자산 카탈로그](A-assets.md)
