# OSSO — 오픈소스 서버·서비스 안전운영 실습 (심화 핸드북)

*Open-Source Secure Operations* 2일 실습 과정의 **심화 핸드북**. 랩에서 체험한 것의 "왜"를 담습니다.

- 📖 **온라인:** https://kreonet.github.io/osso-book
- 🛠 **엔진:** [mdBook](https://rust-lang.github.io/mdBook/) · `main`에 push하면 GitHub Actions가 빌드·배포

## 로컬에서 보기
```bash
# mdBook 설치 (한 번)
cargo install mdbook            # 또는 릴리스 바이너리 내려받기
mdbook serve -o                 # http://localhost:3000
```

## 구조
`src/SUMMARY.md` 가 목차, `src/*.md` 가 각 장. 7개 파트(이미지·방화벽·네트워크·컨테이너·신뢰·접근·AI거버넌스) + 부록.

> WIP — 목차가 먼저 서고 내용이 채워지는 중입니다.
