# 오픈소스 안전운영 실습

**서버와 서비스를 맡게 된 사람을 위한 손으로 익히는 보안**

OSSO(오픈소스 서버·서비스 안전운영 실습) 과정의 교재.

*Open-Source Secure Operations* — 기관 전산담당자·연구실무자를 위한 실습 중심 교재.

- 📖 **온라인:** https://kreonet.github.io/osso-book
- 🗺️ **어디서부터?** 처음부터 읽지 말고 [지도](https://kreonet.github.io/osso-book/map.html)에서 내 상황에 맞는 곳을 고르세요.

## 혼자 공부하기 — AI와 함께

```bash
git clone https://github.com/KREONET/osso-book
cd osso-book
claude        # 또는 쓰는 AI 코딩 도구
```

AI는 `AGENTS.md`를 읽고 시작합니다. "내 상황에 맞는 경로를 같이 고르자"라고 말해 보세요. 오디오북·폰 카드·우리 환경 과제로 바꾸는 프롬프트는 [AI로 내 교재 만들기](https://kreonet.github.io/osso-book/ai/index.html)에 있습니다. 만든 결과물은 `my/`에 쌓이고 커밋되지 않습니다.

## 구조

| 경로 | 내용 |
| --- | --- |
| `src/map.md` | 지도 — 상황별 시작점, 줄기, 코스, 실습별 필요 환경 |
| `src/labs/` | 실습 ①~⑭ (겪어 보기) |
| `src/01~07-*.md` | 심화 Part 1~7 (왜 그런가) |
| `src/A~F-*.md` | 부록 |
| `src/ai/` | AI 변형 프롬프트 |
| `src/llms.txt` | AI용 목차 (원문 마크다운 링크) |

## 로컬에서 보기

```bash
cargo install mdbook            # 또는 릴리스 바이너리 내려받기
mdbook serve -o                 # http://localhost:3000
```

`main`에 push하면 GitHub Actions가 빌드해 배포합니다.
