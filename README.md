# FNDM 랜딩 페이지

K-POP 포카 교환 플랫폼 — 정식 출시 전 사전 가입 랜딩 페이지.
**프론트엔드 완성. 백엔드 연동 대기 중.**

---

## 📁 파일 구조

```
fndm-site/
├── index.html              # 메인 랜딩 페이지
├── legal.html              # 개인정보처리방침 + 이용약관 (5개 언어)
├── README.md               # 이 문서
├── fndm-logo.png           # 빨간 FNDM 로고 (투명 배경) — 네비게이션 바
├── fndm-logo-black.png     # 검정 FNDM 로고 (투명 배경) — 법적 페이지 hero
├── og-image.png            # Open Graph 미리보기 이미지 (1200x630)
├── favicon.ico             # 멀티사이즈 favicon
├── favicon-32.png          # 32x32
├── favicon-192.png         # 192x192 (Apple touch icon)
├── favicon-512.png         # 512x512 (PWA)
├── photocard-1.jpg ~ 5.jpg # 데모 포카 이미지 (블러 처리됨)
```

---

## 🌐 다국어 (i18n) 시스템

**5개 언어 지원**: 한국어(KOR), 영어(EN), 일본어(JP), 중국어(CN), 스페인어(Esp)

### 작동 방식
- **기본 언어는 영어**
- **첫 방문 시** 브라우저 언어를 감지해서 모달 표시 ("한국에서 오셨군요! 한국어로 변경해드릴까요?")
- 사용자가 선택한 언어는 `localStorage` (`fndm-locale`)에 저장됨
- `index.html`과 `legal.html`이 같은 언어 설정 공유

### 번역 텍스트 위치
모든 UI 텍스트는 `index.html` 하단 `<script>` 안의 `const I18N` 객체에 있어요.
같은 키 구조가 5개 언어에 모두 들어있어요.

### 모달 다시 표시 (테스트용)
URL 뒤에 `?reset` 붙이면 localStorage 초기화되어서 모달 다시 표시:
```
https://www.thefndm.com/?reset
```

---

## 🟢 백엔드 — Supabase 무료 티어 가이드

백엔드는 **Supabase 무료 티어**로 구축할 예정. 신용카드 필요 없음.

### 무료 티어 한도 (사전출시 단계에 충분)
- **500 MB** PostgreSQL DB
- **50,000명** 월 활성 사용자
- **5 GB** 월 대역폭
- **500,000회** Edge Function 호출/월
- 무제한 API 요청

### 1단계 — 프로젝트 생성
1. https://supabase.com 무료 가입 (신용카드 불필요)
2. New project → **지역은 한국 가까운 곳 (Tokyo 또는 Seoul)** 선택
3. DB 패스워드 설정 (꼭 저장)
4. 약 2분 대기 (프로젝트 생성)

### 2단계 — `registrations` 테이블 생성

Supabase SQL 에디터에서 실행:

```sql
create table registrations (
  id              uuid primary key default gen_random_uuid(),
  email           text not null unique,
  artist          text not null,
  artist_other    text,
  country         text not null,
  tier            text[] not null,        -- ['preorder'] 또는 ['preorder', 'beta']
  locale          text not null default 'KOR',
  created_at      timestamptz default now()
);

-- 이메일 검색 빠르게 (UNIQUE 제약으로 자동 생성됨)
create index registrations_tier_idx on registrations using gin (tier);
```

### 3단계 — Row Level Security (RLS) 설정

```sql
alter table registrations enable row level security;

-- 누구나 INSERT 가능 (익명 가입 허용)
create policy "anon insert" on registrations
  for insert
  to anon
  with check (true);

-- SELECT는 차단 (이메일 노출 방지) — 카운트는 RPC 함수로만 제공
```

### 4단계 — 통계 카운터 함수

이메일 노출 없이 카운트만 제공:

```sql
create or replace function get_registration_stats()
returns json
language plpgsql
security definer
as $$
declare
  preorder_n int;
  beta_n int;
begin
  select count(*) into preorder_n from registrations where 'preorder' = any(tier);
  select count(*) into beta_n from registrations where 'beta' = any(tier);
  return json_build_object(
    'preorder_count', preorder_n,
    'beta_count', beta_n
  );
end;
$$;

grant execute on function get_registration_stats() to anon;
```

### 5단계 — 등록 함수 (중복 + 베타 마감 자동 처리)

```sql
create or replace function register_user(
  p_email text,
  p_artist text,
  p_artist_other text,
  p_country text,
  p_tier text[],
  p_locale text
)
returns json
language plpgsql
security definer
as $$
declare
  existing_id uuid;
  beta_n int;
  accepted_tier text[];
  preorder_n int;
begin
  -- ① 이메일 중복 체크 → "이미 팬덤과 매칭이 되신 유저입니다"
  select id into existing_id from registrations where email = p_email;
  if existing_id is not null then
    return json_build_object('ok', false, 'error', 'EMAIL_EXISTS');
  end if;

  -- ② 베타 11,000명 마감 체크
  accepted_tier := p_tier;
  if 'beta' = any(p_tier) then
    select count(*) into beta_n from registrations where 'beta' = any(tier);
    if beta_n >= 11000 then
      -- 베타 제외하고 사전예약만 남기기
      accepted_tier := array_remove(p_tier, 'beta');
      if array_length(accepted_tier, 1) is null then
        accepted_tier := array['preorder'];
      end if;
    end if;
  end if;

  -- ③ DB에 저장
  insert into registrations (email, artist, artist_other, country, tier, locale)
  values (p_email, p_artist, p_artist_other, p_country, accepted_tier, p_locale);

  -- ④ 통계 + 베타 거부 안내 응답
  select count(*) into preorder_n from registrations where 'preorder' = any(tier);
  select count(*) into beta_n from registrations where 'beta' = any(tier);

  if 'beta' = any(p_tier) and not ('beta' = any(accepted_tier)) then
    -- 베타 신청했지만 마감되어 사전예약으로만 등록됨
    return json_build_object(
      'ok', false,
      'error', 'BETA_CAP_REACHED',
      'accepted_tier', accepted_tier,
      'stats', json_build_object('preorder_count', preorder_n, 'beta_count', beta_n)
    );
  end if;

  return json_build_object(
    'ok', true,
    'stats', json_build_object('preorder_count', preorder_n, 'beta_count', beta_n)
  );
end;
$$;

grant execute on function register_user(text, text, text, text, text[], text) to anon;
```

### 6단계 — 프론트엔드 연동 (`index.html`)

`<head>`에 Supabase JS 라이브러리 추가:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

하단 `<script>` 안의 `submitForm()` 함수를 다음과 같이 수정:

```js
// Supabase 클라이언트 초기화 (페이지 로드 시 1회)
const sb = supabase.createClient(
  'https://[YOUR_PROJECT].supabase.co',
  '[YOUR_ANON_PUBLIC_KEY]'   // 프론트에 노출해도 안전 — RLS가 보호함
);

async function submitForm(e) {
  e.preventDefault();
  // ... 기존 검증 코드 그대로 유지 ...

  const dict = (typeof I18N !== 'undefined') ? I18N[currentLocale] : null;

  const payload = {
    p_email: emailInput.value.trim(),
    p_artist: form.querySelector('#artist').value,
    p_artist_other: form.querySelector('#artist-other').value || null,
    p_country: form.querySelector('#country').value,
    p_tier: Array.from(form.querySelectorAll('[name="tier"]:checked')).map(c => c.value),
    p_locale: currentLocale
  };

  const { data, error } = await sb.rpc('register_user', payload);

  // 서버 오류
  if (error) {
    showFormToast((dict && dict['form.error.server']) || 'Something went wrong. Please try again.');
    return false;
  }

  if (!data.ok) {
    // 이메일 중복
    if (data.error === 'EMAIL_EXISTS') {
      showFormToast((dict && dict['form.error.dup']) || 'You are already matched with the fandom.');
      return false;
    }
    // 베타 마감 — 사전예약만 처리됨
    if (data.error === 'BETA_CAP_REACHED') {
      showFormToast((dict && dict['form.error.betacap']) || 'Beta is closed — registered for pre-launch instead.');
      // 그래도 사전예약은 성공이니까 success 화면으로 진행
    }
  }

  // 카운터 업데이트
  if (data.stats) {
    document.getElementById('betaCount').textContent = data.stats.beta_count;
    // 사전예약 카운터도 함께 업데이트
  }

  form.style.display = 'none';
  document.getElementById('form-success').hidden = false;
  return false;
}

// 페이지 로드 시 실시간 통계 가져오기
(async () => {
  const { data } = await sb.rpc('get_registration_stats');
  if (data) {
    document.getElementById('betaCount').textContent = data.beta_count;
    // 사전예약 카운터도 함께 업데이트
    checkBetaCap();   // 실제 베타 카운트로 11,000 마감 체크 재실행
  }
})();
```

### 7단계 — 다국어 안내 메시지 i18n 키 추가

`index.html`의 `I18N` 객체 안 5개 언어 블록 모두에 추가:

```js
// 한국어 (KOR)
'form.error.dup': '이미 팬덤과 매칭이 되신 유저입니다. FNDM 론칭 소식을 가장 먼저 받게 됩니다.',
'form.error.betacap': '베타테스터 모집이 마감되어 사전예약으로 등록되었습니다.',
'form.error.server': '일시적인 오류가 발생했어요. 잠시 후 다시 시도해주세요.',

// 영어 (EN)
'form.error.dup': "You're already matched with the fandom. We'll be the first to send you FNDM launch news.",
'form.error.betacap': 'Beta registration is closed. You have been registered for pre-launch instead.',
'form.error.server': 'Something went wrong. Please try again shortly.',

// 일본어 (JP)
'form.error.dup': 'すでにファンダムとマッチング済みのユーザーです。FNDMローンチの最新情報を最初にお届けします。',
'form.error.betacap': 'ベータテスター募集は終了しました。事前登録として登録されました。',
'form.error.server': '一時的なエラーが発生しました。少し時間をおいて再度お試しください。',

// 중국어 (CN)
'form.error.dup': '您已经与粉丝群匹配。FNDM 上线消息将第一时间送达。',
'form.error.betacap': '内测招募已结束。已为您登记为预约用户。',
'form.error.server': '发生临时错误。请稍后再试。',

// 스페인어 (Esp)
'form.error.dup': 'Ya estás matcheado con el fandom. Serás el primero en recibir las novedades de FNDM.',
'form.error.betacap': 'El registro de beta testers está cerrado. Te registramos para el pre-lanzamiento.',
'form.error.server': 'Algo salió mal. Por favor intentá de nuevo en unos momentos.',
```

### 8단계 — 이메일 알림 (선택사항)

신규 등록 시 관리자에게 알림 보내는 방법:

**방법 A**: Supabase Database Webhooks 사용
- Database → Webhooks → 새 웹훅 생성
- `INSERT` on `registrations` 테이블 트리거
- Slack incoming webhook 또는 이메일 서비스 (Resend, SendGrid 등)로 POST

**방법 B**: Supabase Edge Function 사용
- 등록 시 `pg_notify` 트리거 → Edge Function이 이메일 발송

### 추가 사항
- **anon key는 프론트엔드에 노출해도 안전** — RLS + RPC 함수가 데이터 보호
- **Service Role Key는 절대 프론트엔드에 넣으면 안 됨**
- 관리자 페이지 (모든 등록 조회): Supabase Studio 사용 (Supabase에 기본 제공)
- 무료 티어는 **1주일 미사용 시 자동 일시정지** — https://cron-job.org 같은 서비스로 매일 API 한 번 호출하면 유지 가능

---

## 🔧 백엔드가 처리할 핵심 로직 요약

### A. 베타테스터 11,000명 마감

**프론트엔드 이미 구현됨**:
- `index.html`의 `#betaCount` 카운터 (현재 75 하드코딩)
- `checkBetaCap()` 함수가 11,000 이상이면 베타 체크박스 자동 비활성화 + "마감" 배지 표시

**백엔드가 해야 할 일**:
- 페이지 로드 시 `get_registration_stats()` 호출해서 실제 카운트로 `#betaCount` 업데이트
- 사용자가 베타 신청했지만 이미 11,000명 차있으면 → `BETA_CAP_REACHED` 응답 → 사전예약으로만 등록

### B. 이메일 중복 감지

**프론트엔드 이미 구현됨**:
- 5개 언어로 "이미 팬덤과 매칭이 되신 유저입니다" 메시지 준비됨 (`form.error.dup` 키)
- `showFormToast()` 함수로 다크 모달 토스트 표시

**백엔드가 해야 할 일**:
- 등록 전 이메일 존재 여부 체크
- 이미 있으면 → `EMAIL_EXISTS` 응답
- 프론트엔드가 자동으로 토스트 안내

### C. 통계 카운터 실시간 동기화

**프론트엔드 이미 구현됨**:
- 사전예약 카드 + Proof 섹션에 카운터 2벌 (각각 사전예약 / 베타)

**백엔드가 해야 할 일**:
- `get_registration_stats()` RPC 제공
- 페이지 로드 시 + 등록 성공 시 카운터 업데이트

---

## 📤 등록 폼 필드 명세

폼 위치: `index.html` 약 2710번째 줄 (`<form id="signup-form">`)

| 필드명 | DOM ID | 필수 | 비고 |
|---|---|---|---|
| `email` | `#email` | ✅ | 정규식 검증 적용 |
| `artist` | `#artist` | ✅ | K-POP 그룹 16개 + "기타" 옵션 (ABC순) |
| `artist_other` | `#artist-other` | 조건부 | `artist === 'other'` 일 때만 표시 |
| `country` | `#country` | ✅ | 25개 국가 + 국기 이모지 |
| `tier` | `[name="tier"]` | ✅ (1개 이상) | 체크박스: `preorder` 또는 `beta` |
| `agree` | `#agree` | ✅ | 약관 동의 |

---

## 🎨 디자인 토큰

```css
--bg: #FFFFFF;            /* 배경 */
--ink: #0A0E1A;           /* 메인 텍스트 */
--ink-3: #5A6172;         /* 보조 텍스트 */
--primary: #FF1F3D;       /* FNDM 빨강 */
```

**폰트**:
- **Inter** (영문 디스플레이)
- **Pretendard** (한글 디스플레이+본문)
- **JetBrains Mono** (작은 라벨)

---

## 📧 회사 정보

- **회사명**: 주식회사 컬처코드 (CultureCode Inc.)
- **사업자등록번호**: 158-87-03985
- **개인정보 처리담당자 / 일반 문의**: fndm@culturecode.co.kr (담당자: 임형중)
- **이용약관 관련 회사 문의**: hello@culturecode.co.kr
- **회사 웹사이트**: https://www.culturecode.co.kr
- **목표 도메인**: www.thefndm.com

---

## 🚀 배포 가이드

### 정적 배포 (Vercel / Netlify / Cloudflare Pages)
빌드 단계 없음. 폴더 통째로 업로드만 하면 됨.

### 정식 도메인 연결 후 OG 이미지 처리
현재 메타 태그의 `og:image`는 `https://www.thefndm.com/og-image.png`를 가리킴.
- 정식 도메인 연결되면 자동 작동
- Vercel preview 도메인에서 테스트 중이면 카카오톡 OG가 안 뜰 수 있음
- 카카오톡 캐시 강제 갱신: https://developers.kakao.com/tool/clear/og

### Vercel 캐시 설정 (`vercel.json`)
```json
{
  "headers": [
    {
      "source": "/(.*).(png|jpg|jpeg|ico|svg)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    },
    {
      "source": "/(.*).(html)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=0, must-revalidate" }
      ]
    }
  ]
}
```

---

## 📋 페이지 섹션 구성

### `index.html` (메인 페이지)
1. **NAV** — 스티키 FNDM 로고 + 언어 스위처 (그라데이션 페이드)
2. **HERO** — "포카 교환, 이제 그만 기다려요." 헤드라인
3. **AUTO SWIPE DEMO** — 아이폰 mockup, 자동 스와이프 카드 + "매칭되었습니다!" 배너
4. **PRE-REGISTER CARD** — 카운터 (158/75) + 2개 액션 버튼
5. **PROBLEM** — 4개 페인 포인트 카드
6. **HOW IT WORKS** — 3단계 (스캔 / 스와이프 / 안전 체크)
7. **SOLUTION** — 4축 카드
8. **TRADE LOOP** — 지구본 + 10개 도시 도트 (Seoul, Tokyo, Shanghai, Dubai, LA, NY, Paris, Jakarta, Sydney, São Paulo)
9. **PROOF** — 통계 카드 + 9개 관심사 칩 캐러셀 (드래그 가능)
10. **APP NAV PREVIEW** — 5개 박스 (CONTENT / SHOP / TRADE-NEW / EVENTS / COMMUNITY)
11. **CTA / 가입 폼**
12. **FOOTER** — SNS + 태그라인 + 법적 링크 + 저작권

### `legal.html` (법적 페이지)
- 스티키 nav: 빨간 FNDM 로고 + 뒤로가기 버튼
- 큰 검정 FNDM 로고 + "For the Real Ones." + "Legal"
- 5개 언어 스위처
- 탭 스위처 (개인정보처리방침 ↔ 이용약관)
- URL 해시로 직접 링크: `#privacy` 또는 `#terms`
- 메인과 동일한 푸터 + 사업자등록번호 포함
- 개인정보처리방침 최상단 요약 카드 (수집 항목 / 이용 목적 / 보유 기간)

---

## ✅ 이미 작동하는 기능

- 모든 UI, 애니메이션, 트랜지션
- 5개 언어 i18n + 브라우저 자동 감지 모달
- 폼 클라이언트 검증 (이메일 형식, tier 선택, 동의 체크)
- 가운데 모달 토스트 (네이티브 alert 대체)
- 페이지 간 페이드 트랜지션 (index ↔ legal)
- 메일 아이콘 mailto 자동 언어별 제목
- 친구에게 공유하기 (클립보드 복사)
- "기타" 아티스트 직접 입력 필드
- 25개 국가 선택 (국기 이모지)
- 베타 마감 자동 처리 (프론트엔드 준비됨)
- SEO 메타 + Open Graph + Twitter Card + JSON-LD
- 5개 언어 hreflang
- 모바일 반응형 (모든 섹션)
- 라이트 모드 강제 (다크 모드 오버라이드 방지)

## ⏳ 백엔드 작업 대기 중

- 실제 폼 제출 (현재 정적 success 화면만 표시)
- 이메일 중복 감지 (Supabase RPC `register_user`)
- 베타 11,000명 마감 적용 (Supabase RPC)
- 카운터 실시간 업데이트 (Supabase RPC `get_registration_stats`)
- 관리자용 이메일/슬랙 알림 (선택)

---

마지막 업데이트: 2026년 5월 14일
