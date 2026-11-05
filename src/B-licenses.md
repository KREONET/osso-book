# 부록 B. 오픈소스 라이선스 치트

> **오픈소스 = 공짜가 아니라, 라이선스를 읽고 고른 것.**

도구를 고르기 전에 라이선스 한 줄을 확인하는 습관이 섀도우 IT를 양지로 만드는 첫 걸음입니다.

## 두 묶음 비교

| 구분 | 함정 없음 (권장) | 함정 (주의) |
|---|---|---|
| 라이선스 | GPL/LGPL, MIT, Apache-2.0, BSD | PUEL, Anaconda 약관, SSPL, Docker Desktop 약관 |
| 상업·기관 사용 | 자유롭게 가능 | 조건·유료·재배포 제약이 숨어 있음 |
| 이 과정의 도구 | PVE, KVM·QEMU, Ventoy, Clonezilla, GParted, caddy, Keycloak, miniforge, marimo, Docker Engine, Podman … | VirtualBox Extension Pack, Anaconda 기본 채널, MongoDB/Elasticsearch, Docker Desktop |

## 함정 4종과 대안

| 함정 | 왜 함정인가 | 대체 |
|---|---|---|
| VirtualBox **Extension Pack (PUEL)** | 본체는 GPL이지만 확장팩(USB 2/3, RDP 등)은 개인·평가용만 무료, 기관 사용은 **상용 유료** | PVE/KVM·QEMU, 확장팩 없이 본체만 사용 |
| **Anaconda 기본 채널** | 대형 조직(약 200명 초과)은 상용 ToS 적용 대상 | **miniforge**(conda-forge 채널) |
| MongoDB / Elasticsearch **SSPL** | 서비스로 제공 시 전체 소스 공개 의무. **OSI 비승인**(오픈소스 아님) | PostgreSQL, OpenSearch 등 OSI 승인 라이선스 |
| **Docker Desktop** | 대기업(직원·매출 기준 초과)은 유료 구독 | **Docker Engine** 또는 **Podman** |

## 한 줄 가이드

- 도입 전: `LICENSE` 파일과 "상업적 사용" 문구부터 읽는다.
- "본체는 무료, 확장·채널·데스크톱판은 유료"인 패턴을 의심한다.
- 확신이 없으면 [부록 C](C-adr.md)의 ADR에 "라이선스 검토" 항목으로 남긴다.
- 약관은 바뀝니다. 기준 수치는 도입 시점에 공식 문서로 재확인하세요.

용어: [PUEL · SSPL](F-glossary.md)
