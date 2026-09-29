# 지안이 칭찬스티커

## 프로젝트 구조
- 정적 PWA: `index.html`(HTML/CSS/JS 단일 파일), `sw.js`(서비스워커), `manifest.json`
- 배포: `main` 브랜치 → GitHub Pages 자동 배포
- 버전: `main`에 푸시되면 `.github/workflows/bump-version.yml`이 patch 버전을 자동으로 올리고 `manifest.json`, `index.html`(APP_VERSION), `sw.js`(CACHE_NAME)에 반영한다. 버전을 손으로 올리지 않는다.
- 데이터: `localStorage` 캐시 + Supabase `app_data` 테이블(key/value) 동기화 + Realtime
  - 도장은 하루 한 행 `stamp:YYYY-MM-DD`이고, 월별·목표기간 캘린더가 같은 값을 같이 읽는다.
  - 예전 형식(`stamps:YYYY-MM`, `periodStamps:<id>`)은 `migrateLegacyStamps()`가 하루 단위로 옮기고 지운다.
  - 그 밖의 키: `goal:YYYY-MM`(이번 달 목표), `periods:list`(목표기간 목록), `settings:bgImage`(배경 사진)

## 작업 규칙
- PR만 만들고, 병합은 사용자가 "병합"이라고 할 때만 한다.
- 수정 후 사용자에게 확인을 부탁할 때는 앱 하단 버전 번호로 새 버전이 적용됐는지부터 확인하게 한다.

## 외부 연동(Supabase 등) 작업 체크리스트
실제로 겪은 문제에서 나온 규칙이다. 연동 기능을 추가하거나 고칠 때 반드시 확인한다.

1. **서비스워커는 다른 도메인 요청을 절대 캐시하지 않는다.**
   `sw.js`의 fetch 핸들러는 같은 도메인(앱 파일)만 처리하고 나머지는 그냥 통과시킨다. 앱 파일은 네트워크 우선, 캐시는 오프라인 대비용.
   → 모든 GET을 캐시 우선으로 처리해서 Supabase 조회가 옛날 응답을 돌려줬고, "새로고침하면 도장이 사라짐"의 진짜 원인이었다.
2. **PostgREST(Supabase REST) 주소에 임의의 쿼리 파라미터를 붙이지 않는다.**
   모르는 파라미터는 컬럼 필터로 해석돼 `failed to parse filter` 에러가 난다. 캐시 회피는 `fetch(..., { cache: 'no-store' })`로 한다.
3. **upsert는 `?on_conflict=<기본키 컬럼들>` + `Prefer: resolution=merge-duplicates`로 한다.**
4. **저장 함수는 원격 저장이 끝날 때까지 await 한다.** 저장 중에는 버튼에 "저장 중..."을 표시한다.
5. **여러 기기가 같은 데이터를 고칠 수 있으면 큰 JSON 덩어리를 통째로 덮어쓰지 않는다.**
   가능하면 항목(날짜) 단위로 행을 나눈다. 덩어리를 유지해야 하면 저장 직전에 서버 최신값을 받아 바뀐 항목만 반영한다.
6. **Realtime은 테이블을 `supabase_realtime` publication에 추가해야 이벤트가 온다.** 설정 SQL에 항상 포함한다.
   모바일은 백그라운드에서 연결이 끊기므로 화면 복귀(`visibilitychange`)·네트워크 복귀(`online`) 때 재조회와 재연결을 한다.
7. **사용자에게 주는 설정 SQL은 여러 번 실행해도 에러가 안 나게 만든다.**
   (`create table if not exists`, `drop policy if exists` 후 생성, publication 추가는 중복 예외 처리) — 아래 예시 참고.

### 이 프로젝트의 Supabase 설정 SQL (재실행 가능)
```sql
create table if not exists app_data (
  app_id text not null default 'jian-stamp',
  key text not null,
  value text,
  updated_at timestamptz not null default now(),
  primary key (app_id, key)
);
alter table app_data enable row level security;

drop policy if exists "anon select" on app_data;
drop policy if exists "anon upsert" on app_data;
drop policy if exists "anon update" on app_data;
drop policy if exists "anon delete" on app_data;
create policy "anon select" on app_data for select using (true);
create policy "anon upsert" on app_data for insert with check (true);
create policy "anon update" on app_data for update using (true);
create policy "anon delete" on app_data for delete using (true);

do $$ begin
  alter publication supabase_realtime add table app_data;
exception when duplicate_object then null;
end $$;
```

## 디버깅 원칙
- 같은 버그를 두 번 고쳐도 안 되면 추측 수정을 멈추고, 화면에 진단 정보를 띄우거나 가짜 서버로 재현해서 원인부터 확정한다.
- 원인을 모르겠으면 데이터가 지나가는 경로 전체(서비스워커, 브라우저 캐시, 네트워크 요청, 서버 저장값)를 처음부터 다시 읽는다. 서버 저장값은 Supabase SQL Editor로 직접 조회해서 앱 화면과 비교한다.
- 원인이 확정되고 수정이 확인되면 진단 코드는 지운다.
