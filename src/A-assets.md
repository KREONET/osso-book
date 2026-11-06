# 부록 A. 자산 카탈로그 {#appendix-a}

이 과정에서 쓰고 만든 **자산(저장소·도구·문서)** 의 지도입니다. 필요한 자산의 위치가 기억나지 않을 때 여기서 태그로 찾아보세요. 공개 저장소는 이름으로, 사내 서비스는 **역할만** 적었습니다.

## 공개 저장소·도구 {#a-public}

| 태그 | 저장소/도구 | 역할 |
|---|---|---|
| sadp | [github.com/msjang/ostday26-sadp](https://github.com/msjang/ostday26-sadp) | SADP 4통제지점 · 섀도우 IT(조직이 파악하지 못한 채 운영되는 시스템) 양지화 |
| republock | [github.com/KREONET/republock](https://github.com/KREONET/republock) | 평판 기반 nftables 방화벽(인바운드) |
| caddy-xproxy | [github.com/kreonet/caddy-xproxy](https://github.com/kreonet/caddy-xproxy) | 리버스 프록시(요청을 받아 뒤쪽 서버에 전달하는 서버). `conf.d` opt-in(명시적으로 켠 사이트에만 적용), ACME 인증서, 봇 차단 |
| saml-oidc | [github.com/KREONET/saml-oidc-test-sandbox](https://github.com/KREONET/saml-oidc-test-sandbox) | OIDC/SAML 연동 샌드박스 |
| ufw-docker | [github.com/chaifeng/ufw-docker](https://github.com/chaifeng/ufw-docker) | Docker가 ufw를 우회하는 문제 해결 |
| ohmyzsh | [github.com/kreonet/ohmyzsh](https://github.com/kreonet/ohmyzsh) | 서버 zsh 운영환경 · 자산 태그(TAG1/TAG2/HNUM) |
| miniforge | [github.com/conda-forge/miniforge](https://github.com/conda-forge/miniforge) | conda-forge 기본 설치본(Anaconda 약관 회피) |
| marimo | [github.com/marimo-team/marimo](https://github.com/marimo-team/marimo) | 반응형 파이썬 노트북 |
| adr | [github.com/adr/madr](https://github.com/adr/madr) | MADR(결정 기록) 템플릿 → [부록 C](C-adr.md) |

## 사내 서비스 (호스트명 생략) {#a-internal}

| 태그 | 역할 |
|---|---|
| isetup | 서버 초기 세팅 부트스트랩 서비스 |
| demo-zone | 실습·시연용 격리 환경 |
| chat | 사내 LLM 백엔드(채팅 API) |
| ns5 | 사내 DNS 서비스 |

## 운영 문서 (이 책 안) {#a-docs}

| 태그 | 내용 | 이 책의 위치 |
|---|---|---|
| pve-guide | Proxmox VE 설치·구성·활용 | [부록 H](H-pve.md) |
| pfsense-guide | pfSense 방화벽 운영 | [Part 3 §3.3](03-network.md), [실습⑦](labs/lab-07.md) |
| access-control | 네트워크·시스템 접근제어(NAC·802.1X 등) | [부록 I](I-access-control.md), [Part 6 §6.5](06-access.md) |
| workstation | 내 PC 작업 환경(터미널·SSH 클라이언트·셸) | [부록 J](J-workstation.md) |
| delivery | 서버 납품·인수 | [부록 K](K-delivery.md) |
| packages | 패키지 관리자 | [부록 G](G-packages.md) |

> 운영 문서의 내용은 모두 이 책 안에 옮겨 두었습니다. 옮겨 온 원래 문서와 책 위치의 대응은 [부록 E](E-references.md)에 있습니다.

관련: [부록 B 라이선스](B-licenses.md) · [부록 D 치트시트](D-cheatsheet.md) · [부록 F 용어집](F-glossary.md)
