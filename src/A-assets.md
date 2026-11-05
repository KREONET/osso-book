# 부록 A. 자산 카탈로그

이 과정에서 쓰고 만든 **자산(저장소·도구·문서)** 의 지도입니다. "이게 어디 있더라?"가 생길 때 여기서 태그로 찾으세요. 공개 저장소는 이름으로, 사내 서비스는 **역할만** 적었습니다.

## 공개 저장소·도구

| 태그 | 저장소/도구 | 역할 |
|---|---|---|
| sadp | [github.com/msjang/ostday26-sadp](https://github.com/msjang/ostday26-sadp) | SADP 4통제지점 · 섀도우 IT 양지화 |
| republock | [github.com/KREONET/republock](https://github.com/KREONET/republock) | 평판 기반 nftables 방화벽(인바운드) |
| caddy-xproxy | [github.com/kreonet/caddy-xproxy](https://github.com/kreonet/caddy-xproxy) | 리버스 프록시. `conf.d` opt-in, ACME 인증서, 봇 차단 |
| saml-oidc | [github.com/KREONET/saml-oidc-test-sandbox](https://github.com/KREONET/saml-oidc-test-sandbox) | OIDC/SAML 연동 샌드박스 |
| ufw-docker | [github.com/chaifeng/ufw-docker](https://github.com/chaifeng/ufw-docker) | Docker가 ufw를 우회하는 문제 해결 |
| ohmyzsh | [github.com/kreonet/ohmyzsh](https://github.com/kreonet/ohmyzsh) | 서버 zsh 운영환경 · 자산 태그(TAG1/TAG2/HNUM) |
| miniforge | [github.com/conda-forge/miniforge](https://github.com/conda-forge/miniforge) | conda-forge 기본 설치본(Anaconda 약관 회피) |
| marimo | [github.com/marimo-team/marimo](https://github.com/marimo-team/marimo) | 반응형 파이썬 노트북 |
| adr | [github.com/adr/madr](https://github.com/adr/madr) | MADR(결정 기록) 템플릿 → [부록 C](C-adr.md) |

## 사내 서비스 (호스트명 생략)

| 태그 | 역할 |
|---|---|
| isetup | 서버 초기 세팅 부트스트랩 서비스 |
| demo-zone | 실습·시연용 격리 환경 |
| chat | 사내 LLM 백엔드(채팅 API) |
| ns5 | 사내 DNS 서비스 |

## 사내 위키 문서

| 태그 | 내용 |
|---|---|
| pve-guide | Proxmox VE 운영 가이드 |
| pfsense-guide | pfSense 방화벽 운영 가이드 |
| access-control | 접근통제(NAC·802.1X 등) 문서 |

> 위키는 사내 열람용이라 공개본에는 제목만 둡니다. → [부록 E](E-references.md)

관련: [부록 B 라이선스](B-licenses.md) · [부록 D 치트시트](D-cheatsheet.md) · [부록 F 용어집](F-glossary.md)
