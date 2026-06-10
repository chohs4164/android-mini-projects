# keybuddy

다나와 크롤러(`output/keyboards.json`)를 입력으로, 자연어/단계별 질문에 맞춰 키보드를
추천해 주는 웹 서비스입니다.

브라우저에서 OpenAI API를 직접 호출하지 않고, Supabase Edge Function이 서버사이드에서
OpenAI를 호출합니다. Edge Function에서 먼저 후보를 25개 이하로 압축한 뒤 추천 품질을
위해 `gpt-5.4` 모델에 넘깁니다.

## 구조

```text
keybuddy/
  frontend/                         Vite + React + TS + Tailwind
    src/
      App.tsx                       화면(홈 / 단계별 질문 / 결과)
      lib/recommend.ts              Supabase Edge Function 호출
      data/keyboards.json           크롤링 데이터 사본(프론트 표시용)
      types.ts

  supabase/
    .gitignore                      로컬 secret 파일 제외
    config.toml                     recommend 함수 JWT 검증 설정
    functions/
      recommend/
        index.ts                    OpenAI 호출 + 후보 압축 + 결과 매핑
        keyboards.json              추천 후보 카탈로그
```

## 요청 흐름

```text
React 브라우저
  -> Supabase Edge Function /recommend
  -> OPENAI_API_KEY secret 읽기
  -> 후보 25개 이하로 압축
  -> OpenAI gpt-5.4 모델 호출
  -> catalog index 기반 추천 JSON 반환
  -> 프론트가 결과 렌더링
```

브라우저에는 `OPENAI_API_KEY`가 내려가지 않습니다.
Supabase publishable key(`sb_publishable_...`)는 JWT가 아니므로 `Authorization` 헤더가
아니라 `apikey` 헤더로 보냅니다. `recommend` 함수는 공개 추천 엔드포인트라
`supabase/config.toml`에서 `verify_jwt = false`로 설정합니다.

## 실행 방식 선택

keybuddy는 프론트엔드만으로는 추천 기능이 완전히 동작하지 않습니다. 추천 버튼을
누르면 Supabase Edge Function이 OpenAI API를 대신 호출하기 때문입니다.

다른 개발자가 실행할 때는 아래 둘 중 하나를 선택합니다.

| 방식 | 필요한 값 | OpenAI API 키 필요 여부 | 사용 상황 |
| --- | --- | --- | --- |
| 기존 Supabase 추천 API 사용 | `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` | 로컬에는 필요 없음 | 팀원이 이미 배포한 추천 API를 같이 사용할 때 |
| 내 Supabase에 직접 배포 | Supabase 프로젝트, `OPENAI_API_KEY` secret | Supabase secret으로 필요 | 각자 독립 환경을 만들 때 |

`OPENAI_API_KEY`는 절대로 프론트엔드 `.env.local`에 넣지 않습니다.

## 로컬 준비

### 1. 프론트 설정

```bash
cd impl/keybuddy/frontend
cp .env.local.example .env.local
npm install
npm run dev
```

`.env.local`에는 Supabase 프로젝트의 공개 설정값을 채웁니다. 이 두 값은 브라우저에
노출 가능한 값입니다. 단, `.env.local` 파일 자체는 개인 로컬 설정 파일이므로
커밋하지 않습니다.

```env
VITE_SUPABASE_URL=https://your-project-ref.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-anon-key
```

현재 팀 Supabase 프로젝트를 같이 쓰는 경우에는 팀원이 공유한 publishable key를
`VITE_SUPABASE_ANON_KEY`에 넣습니다. Supabase의 새 키 형식에서는
`sb_publishable_...` 형태일 수 있습니다.

로컬 Edge Function을 직접 붙일 때만 아래 값을 추가합니다.

```env
VITE_SUPABASE_RECOMMEND_URL=http://127.0.0.1:54321/functions/v1/recommend
```

이 값이 없으면 프론트는 아래 주소로 배포된 함수를 호출합니다.

```text
${VITE_SUPABASE_URL}/functions/v1/recommend
```

### 2. 기존 Supabase 추천 API를 사용하는 경우

이미 팀 Supabase에 `recommend` 함수가 배포되어 있다면 여기까지만 하면 됩니다.

```bash
cd impl/keybuddy/frontend
npm run dev
```

브라우저에서 Vite가 출력한 주소로 접속합니다.

```text
http://localhost:5173/
```

이 방식에서는 로컬 컴퓨터에 OpenAI API 키가 없어도 됩니다. OpenAI 키는 Supabase
Edge Function secret에만 저장되어 있습니다.

### 3. 내 Supabase에 직접 배포하는 경우

아래 과정은 Supabase 프로젝트를 새로 만들거나, 본인 Supabase 프로젝트로 추천 API를
직접 배포할 때만 필요합니다.

#### Supabase CLI 설정

```bash
npm install -g supabase
supabase login
```

프로젝트 연결:

```bash
cd impl/keybuddy
supabase link --project-ref your-project-ref
```

`project-ref`는 Supabase 프로젝트 URL의 앞부분입니다.

```text
https://abcdefghijk.supabase.co
        ^^^^^^^^^^^
```

#### OpenAI secret 등록

OpenAI API 키는 프론트 `.env.local`에 쓰지 않고, Supabase secret으로만 등록합니다.

```bash
cd impl/keybuddy
supabase secrets set OPENAI_API_KEY=sk-...
supabase secrets set OPENAI_MODEL=gpt-5.4
```

#### Edge Function 배포

```bash
cd impl/keybuddy
supabase functions deploy recommend --project-ref your-project-ref --use-api
```

배포 후 프론트의 `.env.local`을 본인 Supabase 프로젝트 값으로 맞춥니다.

```env
VITE_SUPABASE_URL=https://your-project-ref.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-publishable-key
```

## 로컬 실행

프론트:

```bash
cd impl/keybuddy/frontend
npm run dev
```

Edge Function 로컬 실행:

```bash
cd impl/keybuddy
supabase functions serve recommend --env-file supabase/functions/.env.local
```

로컬 테스트용 `supabase/functions/.env.local` 예시:

```env
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-5.4
```

이 파일은 `supabase/.gitignore`로 제외됩니다. 로컬 Edge Function을 쓰는 경우에만
프론트 `.env.local`에 아래 값을 추가합니다.

```env
VITE_SUPABASE_RECOMMEND_URL=http://127.0.0.1:54321/functions/v1/recommend
```

## 배포

프론트 빌드:

```bash
cd impl/keybuddy/frontend
npm run build
```

Edge Function 배포:

```bash
cd impl/keybuddy
supabase functions deploy recommend
```

새 publishable key(`sb_publishable_...`)를 쓰는 경우 JWT 검증 설정이 반영되어야 하므로,
문제가 있으면 아래처럼 project ref와 API 배포 옵션을 한 줄로 명시합니다.

```bash
supabase functions deploy recommend --project-ref your-project-ref --use-api
```

현재 프로젝트라면 아래처럼 실행합니다.

```bash
supabase functions deploy recommend --project-ref kzgrduvwwoflybrqayyk --use-api
```

프론트 정적 배포는 Supabase Hosting이 아니라 Vercel, Netlify, GitHub Pages 같은 정적
호스팅을 사용하면 됩니다. Supabase는 추천 API 역할만 담당합니다.

정적 배포 환경에도 아래 두 환경변수는 반드시 설정해야 합니다.

```env
VITE_SUPABASE_URL=https://your-project-ref.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-publishable-key
```

`OPENAI_API_KEY`는 정적 배포 환경변수에 넣지 않습니다.

## 문제 해결

### `VITE_SUPABASE_URL이 설정되지 않았습니다.`

`impl/keybuddy/frontend/.env.local`이 없거나 값이 비어 있는 상태입니다.

```bash
cd impl/keybuddy/frontend
cp .env.local.example .env.local
```

그 다음 `.env.local`에 아래 값을 채웁니다.

```env
VITE_SUPABASE_URL=https://your-project-ref.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-publishable-key
```

Vite는 실행 중 환경변수 변경을 자동으로 다시 읽지 못할 수 있으므로, 수정 후
`npm run dev`를 다시 실행합니다.

### `UNAUTHORIZED_INVALID_JWT_FORMAT`

Supabase publishable key를 `Authorization` 헤더에 넣거나, Edge Function의 JWT 검증
설정이 배포 환경에 반영되지 않았을 때 볼 수 있는 오류입니다. 이 프로젝트의 프론트는
publishable key를 `apikey` 헤더로 보냅니다. 함수 배포 시 아래 명령으로
`supabase/config.toml`의 `verify_jwt = false` 설정까지 반영합니다.

```bash
cd impl/keybuddy
supabase functions deploy recommend --project-ref your-project-ref --use-api
```

### OpenAI quota 또는 billing 오류

프론트 설정 문제가 아니라 Supabase Edge Function에 등록된 `OPENAI_API_KEY`의 계정
상태 문제입니다. OpenAI Platform에서 결제/한도 상태를 확인하거나, Supabase secret의
키를 정상 키로 다시 등록해야 합니다.

```bash
cd impl/keybuddy
supabase secrets set OPENAI_API_KEY=sk-...
```

### `npm run dev`에서 `Missing script: "dev"`가 뜨는 경우

명령을 저장소 루트에서 실행한 것입니다. 프론트 폴더로 이동한 뒤 실행합니다.

```bash
cd impl/keybuddy/frontend
npm run dev
```

## 데이터 갱신

상위 크롤러를 다시 돌린 뒤 결과를 프론트와 Supabase 함수 양쪽에 복사합니다. 두
`keyboards.json`이 달라지면 프론트에 보이는 카탈로그와 Edge Function 추천 후보가
엇갈릴 수 있으므로 항상 함께 갱신합니다.

```bash
cd impl
python3 crawl.py
cd keybuddy/frontend
npm run sync:data
```

## 보안/비용 메모

- 프론트에 `VITE_OPENAI_API_KEY` 같은 값을 두지 않습니다.
- `VITE_` 환경변수는 브라우저 번들에 포함됩니다.
- `OPENAI_API_KEY`는 Supabase secret으로만 저장합니다.
- 기본 모델은 추천 품질을 고려해 `gpt-5.4`로 설정합니다.
- Edge Function은 LLM 호출 전에 후보를 25개 이하로 줄여 입력 토큰을 줄입니다.
- `recommend` 함수에는 IP 기준 1분 10회 best-effort rate limit을 둡니다.
- `recommend` 함수는 공개 엔드포인트이므로 운영 시 Supabase Dashboard의 Edge
  Functions rate limit 또는 별도 인증/사용량 제한을 반드시 설정합니다.
- 공개 서비스로 운영할 때는 로그인, 캐싱, 사용자별 사용량 로깅을 추가하는 것이 좋습니다.
