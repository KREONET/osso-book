# Part 2. 방화벽 — 인·아웃바운드 양방향

> 담당 랩 ③④⑤⑥. 이 파트의 2.2·2.4는 분석노트가 거의 초안.

## 2.1 두 사고방식 — ufw vs firewalld(zone)
> 📝 작성 예정.

## 2.2 Docker가 방화벽을 건너뛰는 이유 — DNAT→FORWARD, nftables 체인 3단계
> 📝 분석노트(ufw·nftables)에서 이식 예정.

## 2.3 ufw-docker와 DOCKER-USER 체인
> 📝 작성 예정.

## 2.4 firewalld에서는 — egress=docker policy, 컨테이너 포트 기준
> 📝 분석노트(firewalld·zone)에서 이식 예정.

## 2.5 인바운드 선차단 — 평판 기반 nftables
> 📝 작성 예정.

## 2.6 아웃바운드 화이트리스팅 — ufw default deny outgoing + squid
> 📝 작성 예정.

## 2.7 egress 통제의 네 고도 — 호스트 → 컨테이너 → 망 경계 → AI
이 책 전체를 꿰는 축.
> 📝 작성 예정.
