# 받아쓰기 공책

## 프로젝트 구조
- 정적 PWA: `index.html`(HTML/CSS/JS 단일 파일), `sw.js`(서비스워커), `manifest.json`
- 배포: `main` 브랜치 → GitHub Pages 자동 배포
- 버전: `main`에 푸시되면 `.github/workflows/bump-version.yml`이 버전을 자동으로 올리고(patch와 minor는 10이 되면 윗자리로 올라간다: 1.0.9 → 1.1.0, 1.9.9 → 2.0.0) `manifest.json`, `index.html`(APP_VERSION), `sw.js`(CACHE_NAME)에 반영한다. 버전을 손으로 올리지 않는다.
- 데이터: 서버 없이 `localStorage`만 쓴다. 키: `dictation-settings`(설정), `dictation-items`(고친 문장), `dictation-best`(최고 점수), `dictation-wrong`(틀린 문제), `dictation-stats`(급수별 기록)
- 문제 원문은 `index.html`의 `DATA`(1~10급, 급수당 10문장)이고, 사용자가 설정에서 고친 문장은 `dictation-items`가 덮어쓴다.

## 작업 규칙
- PR만 만들고, 병합은 사용자가 "병합"이라고 할 때만 한다.
- 수정 후 사용자에게 확인을 부탁할 때는 앱 하단 버전 번호로 새 버전이 적용됐는지부터 확인하게 한다.
- 서비스워커는 다른 도메인 요청(Google Fonts 등)을 캐시하지 않는다. 앱 파일만 네트워크 우선으로 받고, 캐시는 오프라인 대비용이다.
- 밝기(밝게/어둡게/기기 설정)와 파스텔 색상은 JS가 `:root`의 색 토큰을 덮어써서 적용한다. 새 색을 쓸 때는 하드코딩하지 말고 토큰(`--accent`, `--accent-ink`, `--line` 등)을 쓴다.

## 디버깅 원칙
- 같은 버그를 두 번 고쳐도 안 되면 추측 수정을 멈추고, 화면에 진단 정보를 띄워서 원인부터 확정한다. 원인이 확정되고 수정이 확인되면 진단 코드는 지운다.
