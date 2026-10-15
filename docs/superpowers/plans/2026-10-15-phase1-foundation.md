# 1단계: 기반 (프로젝트·DB·권한·로그인) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 로그인한 관리자·엔지니어가 역할에 맞는 화면을 보고, PRD의 모든 테이블이 DB 권한(RLS)과 이슈 상태 규칙까지 갖춘 상태로 Vercel에 배포된다.

**Architecture:** Next.js(App Router) 서버 컴포넌트와 Server Action이 `@supabase/ssr` 클라이언트로 Supabase를 직접 호출한다. 권한·코드 채번·이슈 상태 규칙·처리 기록은 모두 Postgres(RLS, 트리거)에서 강제해 화면·엑셀 가져오기·API 어디서 들어오든 같은 규칙이 적용된다. 서비스(secret) 키는 사용자 초대와 테스트에서만 서버 측으로 사용한다.

**Tech Stack:** Next.js 16 · React 19 · TypeScript · Tailwind CSS v4 · shadcn/ui · Supabase (Postgres, Auth) · `@supabase/ssr` · zod · Vitest · Playwright · pnpm

**Spec:** `prd.md`

## 전체 단계 (이 문서는 1단계)

| 단계 | 결과물 | 계획 문서 |
|---|---|---|
| **1. 기반** | 프로젝트, 전체 스키마, RLS, 이슈 규칙 트리거, 로그인, 레이아웃, 사용자 관리, 배포 | 이 문서 |
| 2. 고객사 | 고객사 목록·상세(탭), 설치환경·고객담당자 CRUD, 제품 관리 | 2단계 시작 전 작성 |
| 3. 이슈 | 이슈 목록·상세·등록, 배정, 상태 변경, 처리 기록·코멘트, 유사 이슈 | 3단계 시작 전 작성 |
| 4. 점검·홈 | 점검 목록·등록, 후속 이슈 연결, 홈(내 업무·관리자 현황) | 4단계 시작 전 작성 |
| 5. 엑셀 가져오기 | 업로드→검증 미리보기→트랜잭션 저장, 빈 양식 다운로드 | 5단계 시작 전 작성 |

## Global Constraints

- DB는 **운영 Supabase 프로젝트 하나**만 사용한다 (사용자 결정). `supabase db reset`, `truncate`, 조건 없는 `delete` 를 절대 실행하지 않는다.
- 테스트가 만드는 데이터는 이름이 `[TEST]` 로 시작하고, 테스트 계정 이메일은 `test-<무작위>@example.com` 형식이다. 각 테스트 파일은 `afterAll` 에서 자기가 만든 것을 지운다.
- 서비스 키(`SUPABASE_SECRET_KEY`)는 `server-only` 모듈과 `tests/`, `scripts/` 에서만 import 한다. `NEXT_PUBLIC_` 접두어를 붙이지 않는다.
- 선택값은 DB에 영문 코드로 저장하고 화면에는 `src/lib/labels.ts` 의 한글 라벨로 표시한다.
- 모든 public 테이블은 RLS를 켠다. 정책은 `to authenticated` 로만 만든다 (anon 접근 없음).
- 계산 가능한 값(계약상태, 처리시간)은 컬럼으로 저장하지 않는다.
- 내부 키는 `uuid`, 사람이 읽는 코드는 `code` 컬럼 (`ENG-001`, `CUS-001`, `CON-001`, `INS-001`: 3자리 / `ISS-0001`, `INSP-0001`: 4자리, 넘치면 자리수 늘어남).
- 화면 문구는 한국어. 코드 스타일은 create-next-app 기본(큰따옴표, 세미콜론).
- 회원가입 없음. 계정은 관리자 초대 또는 `scripts/create-admin.ts` 로만 생성.

## Review Focus

1. **비활성화된 사용자가 이미 로그인된 세션으로 접속** → 어떤 데이터도 보이지 않고 로그인 화면으로 보내지며 "비활성화된 계정" 안내가 보여야 한다. (Task 3 RLS 테스트, Task 7 E2E)
2. **관리자가 자기 자신의 역할을 낮추거나 비활성화** → 관리자가 아무도 없는 상태를 막기 위해 거부되어야 한다. (Task 6 `selfChangeError` 테스트)
3. **이미 등록된 이메일로 다시 초대** → 영문 오류 원문이 아니라 "이미 등록된 이메일입니다." 로 안내되어야 한다. (Task 6 `inviteErrorMessage` 테스트)
4. **메일 링크의 `next` 파라미터에 외부 주소(`//evil.com`, `https://…`)** → 외부로 이동하지 않고 `/` 로 가야 한다. (Task 5 `safeNextPath` 테스트)
5. **코드 번호가 자리수를 넘음 (`CUS-999` 다음)** → `CUS-100` 처럼 잘리지 않고 `CUS-1000` 이 되어야 한다. (Task 2 테스트)

---

## 파일 구조

```
.
├── prd.md, 기술지원_더미데이터.xlsx, docs/          (기존)
├── .env.example                      환경변수 목록 (값 없음)
├── vitest.config.ts                  unit / db 두 프로젝트
├── playwright.config.ts
├── supabase/
│   ├── config.toml                   supabase init 생성
│   └── migrations/
│       ├── 20261015000001_schema.sql        enum, 테이블, 코드 채번, updated_at
│       ├── 20261015000002_auth_rls.sql      권한 함수, 프로필 자동 생성, RLS
│       └── 20261015000003_issue_rules.sql   이슈 상태 규칙, 처리 기록 트리거
├── scripts/
│   ├── create-admin.ts               첫 관리자 계정 생성 (사용자가 직접 실행)
│   └── cleanup-test-data.ts          남은 테스트 데이터 정리
├── src/
│   ├── proxy.ts                      세션 갱신 + 비로그인 리다이렉트
│   ├── app/
│   │   ├── layout.tsx                (생성됨) lang="ko" 로 수정
│   │   ├── login/{page.tsx, login-form.tsx, actions.ts}
│   │   ├── forgot-password/{page.tsx, forgot-form.tsx, actions.ts}
│   │   ├── set-password/{page.tsx, set-password-form.tsx, actions.ts}
│   │   ├── auth/confirm/route.ts     메일 링크 토큰 확인
│   │   ├── auth/signout/route.ts
│   │   └── (app)/
│   │       ├── layout.tsx            로그인·활성 확인 + 사이드바
│   │       ├── page.tsx              홈 (4단계에서 교체)
│   │       └── admin/users/{page.tsx, invite-form.tsx, user-row.tsx, actions.ts}
│   ├── components/
│   │   ├── ui/                       shadcn 생성
│   │   └── app-sidebar.tsx
│   └── lib/
│       ├── labels.ts (+test)         영문 코드 ↔ 한글 라벨
│       ├── nav.ts (+test)            역할별 메뉴
│       ├── action-state.ts           Server Action 결과 타입
│       ├── auth/paths.ts (+test)     공개 경로, 안전한 next 경로
│       ├── auth/profile.ts           현재 사용자 프로필, requireAdmin
│       ├── validation/user.ts (+test) 초대·비밀번호 검증, 오류 메시지
│       └── supabase/{server.ts, proxy.ts, admin.ts, database.types.ts}
└── tests/
    ├── support/accounts.ts           테스트 계정 생성·삭제 (db, e2e 공용)
    ├── db/{codes, rls, issues}.test.ts
    └── e2e/{global-setup.ts, global-teardown.ts, auth.spec.ts}
```

---

### Task 1: 프로젝트 생성과 기본 도구

**Files:**
- Create: Next.js 프로젝트 전체 (create-next-app), `vitest.config.ts`, `.env.example`, `src/lib/labels.ts`
- Modify: `.gitignore`, `package.json` (scripts), `src/app/layout.tsx`
- Test: `src/lib/labels.test.ts`

**Interfaces:**
- Produces: `labelOf(map, code): string`, 라벨 맵 `ROLE`, `GRADE`, `INSPECTION_CYCLE`, `ISSUE_CHANNEL`, `ISSUE_CATEGORY`, `ISSUE_PRIORITY`, `ISSUE_STATUS`, `ISSUE_EVENT_TYPE`, `BACKUP_STATUS`, `INSPECTION_RESULT`. npm scripts `test`, `test:db`, `test:e2e`, `db:push`, `db:types`.

- [ ] **Step 1: git 저장소 만들고 기존 문서 커밋**

```bash
cd ~/Desktop/통합기술지원관리시스템
git init -b main
git add prd.md 기술지원_더미데이터.xlsx docs
git commit -m "docs: PRD, 더미 데이터, 1단계 구현 계획"
```

- [ ] **Step 2: Next.js 프로젝트 생성 (임시 폴더에 만든 뒤 옮김 — 폴더에 기존 파일이 있어 직접 생성 불가)**

```bash
pnpm create next-app@16 web-tmp --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-pnpm --disable-git --yes
rsync -a web-tmp/ ./
rm -rf web-tmp
pnpm install
```

확인: `ls src/app` 에 `layout.tsx`, `page.tsx` 가 있다. `src/app/page.tsx` 는 Task 6에서 `(app)/page.tsx` 로 대체하므로 이 단계에서 삭제한다:

```bash
rm src/app/page.tsx
```

- [ ] **Step 3: 의존성과 shadcn/ui 설치**

```bash
pnpm add @supabase/supabase-js @supabase/ssr zod server-only
pnpm add -D vitest vite tsx @playwright/test
pnpm dlx shadcn@latest init -d
pnpm dlx shadcn@latest add button input label card
pnpm exec playwright install chromium
```

- [ ] **Step 4: `.gitignore` 에 항목 추가**

create-next-app 의 `.gitignore` 에는 `.env*` 가 있다. 그 아래에 추가:

```gitignore
!.env.example

# supabase
supabase/.temp
supabase/.branches

# e2e
tests/e2e/.users.json
/test-results
/playwright-report
```

- [ ] **Step 5: `.env.example` 작성**

```dotenv
# Supabase 대시보드 > Project Settings > API Keys
NEXT_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=
# 서버 전용. 절대 NEXT_PUBLIC_ 붙이지 말 것
SUPABASE_SECRET_KEY=
```

사용자에게 요청: `.env.example` 을 `.env.local` 로 복사하고 운영 Supabase 값 3개를 채워 달라고 한다. (Claude는 키 값을 입력하지 않는다.)

- [ ] **Step 6: `vitest.config.ts` 작성**

```ts
import path from "node:path";
import { loadEnv } from "vite";
import { defineConfig } from "vitest/config";

export default defineConfig({
  resolve: { alias: { "@": path.resolve(__dirname, "src") } },
  test: {
    env: loadEnv("test", process.cwd(), ""),
    projects: [
      {
        extends: true,
        test: { name: "unit", include: ["src/**/*.test.ts"] },
      },
      {
        extends: true,
        test: {
          name: "db",
          include: ["tests/db/**/*.test.ts"],
          fileParallelism: false,
          testTimeout: 20_000,
          hookTimeout: 60_000,
        },
      },
    ],
  },
});
```

- [ ] **Step 7: `package.json` scripts 추가**

`"scripts"` 에 다음을 추가 (기존 `dev`, `build`, `start`, `lint` 유지):

```json
"test": "vitest run --project unit",
"test:db": "vitest run --project db",
"test:e2e": "playwright test",
"db:push": "supabase db push",
"db:types": "supabase gen types typescript --linked --schema public > src/lib/supabase/database.types.ts"
```

- [ ] **Step 8: 실패하는 테스트 작성 — `src/lib/labels.test.ts`**

```ts
import { describe, expect, it } from "vitest";
import { ISSUE_CHANNEL, ISSUE_STATUS, labelOf } from "./labels";

describe("labelOf", () => {
  it("영문 코드를 한글 라벨로 바꾼다", () => {
    expect(labelOf(ISSUE_STATUS, "in_progress")).toBe("처리중");
    expect(labelOf(ISSUE_CHANNEL, "inspection")).toBe("정기점검 중 발견");
  });

  it("모르는 코드는 그대로 돌려준다", () => {
    expect(labelOf(ISSUE_STATUS, "unknown")).toBe("unknown");
  });

  it("이슈 상태 5개가 PRD 순서대로 정의되어 있다", () => {
    expect(Object.keys(ISSUE_STATUS)).toEqual([
      "received",
      "assigned",
      "in_progress",
      "done",
      "on_hold",
    ]);
  });
});
```

- [ ] **Step 9: 실패 확인**

Run: `pnpm test`
Expected: FAIL — `Failed to resolve import "./labels"`

- [ ] **Step 10: `src/lib/labels.ts` 구현**

```ts
export const ROLE = { admin: "관리자", engineer: "엔지니어" } as const;

export const GRADE = { A: "A", B: "B", C: "C" } as const;

export const INSPECTION_CYCLE = {
  monthly: "월 1회",
  quarterly: "분기 1회",
  semiannual: "반기 1회",
} as const;

export const ISSUE_CHANNEL = {
  phone: "전화",
  email: "이메일",
  messenger: "메신저",
  inspection: "정기점검 중 발견",
} as const;

export const ISSUE_CATEGORY = {
  outage: "장애",
  bug: "버그",
  question: "문의",
  config: "설정변경",
  feature: "기능요청",
} as const;

export const ISSUE_PRIORITY = {
  urgent: "긴급",
  high: "높음",
  normal: "보통",
  low: "낮음",
} as const;

export const ISSUE_STATUS = {
  received: "접수",
  assigned: "배정",
  in_progress: "처리중",
  done: "완료",
  on_hold: "보류",
} as const;

export const ISSUE_EVENT_TYPE = {
  created: "등록",
  status_changed: "상태 변경",
  assigned: "담당자 변경",
  comment: "코멘트",
  edited: "내용 수정",
} as const;

export const BACKUP_STATUS = { ok: "정상", failed: "실패" } as const;

export const INSPECTION_RESULT = {
  normal: "정상",
  warning: "주의",
  critical: "이상",
} as const;

export function labelOf(map: Record<string, string>, code: string): string {
  return map[code] ?? code;
}
```

- [ ] **Step 11: 통과 확인**

Run: `pnpm test`
Expected: PASS (3 tests)

- [ ] **Step 12: `src/app/layout.tsx` 의 `<html lang="en">` 을 `<html lang="ko">` 로, `metadata.title` 을 `"기술지원 통합관리"` 로 바꾼다.**

- [ ] **Step 13: 빌드 확인**

Run: `pnpm build`
Expected: 성공. (루트 페이지가 없어 `/` 는 404 — Task 6에서 생김)

- [ ] **Step 14: Commit**

```bash
git add -A
git commit -m "chore: Next.js 프로젝트, 테스트 도구, 라벨 모듈"
```

---

### Task 2: Supabase 연결과 기본 스키마

**Files:**
- Create: `supabase/config.toml` (supabase init), `supabase/migrations/20261015000001_schema.sql`, `src/lib/supabase/database.types.ts` (생성), `tests/db/codes.test.ts`

**Interfaces:**
- Consumes: Task 1 scripts `db:push`, `db:types`, `.env.local`
- Produces: 테이블 `profiles`, `customers`, `customer_contacts`, `products`, `installations`, `issues`, `issue_events`, `inspections` (컬럼은 PRD 5.2와 동일). DB 함수 `assign_code()` (트리거용, 인자: 접두어, 자리수), `set_updated_at()`. TS 타입 `Database` (`src/lib/supabase/database.types.ts`).

- [ ] **Step 1: Supabase CLI 연결 (사용자가 직접 실행 — 로그인·DB 비밀번호 입력 필요)**

```bash
supabase init
supabase login
supabase link --project-ref <운영 프로젝트 ref>
```

`supabase init` 이 VS Code 설정 생성 여부를 물으면 No.

- [ ] **Step 2: 실패하는 테스트 작성 — `tests/db/codes.test.ts`**

이 테스트는 서비스 키로 직접 접근한다 (RLS 정책은 Task 3).

```ts
import { createClient } from "@supabase/supabase-js";
import { afterAll, describe, expect, it } from "vitest";

const admin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SECRET_KEY!,
  { auth: { persistSession: false, autoRefreshToken: false } },
);

const created: string[] = [];

async function insertCustomer(fields: { name: string; code?: string }) {
  const { data, error } = await admin
    .from("customers")
    .insert(fields)
    .select("id, code")
    .single();
  if (error) throw error;
  created.push(data.id);
  return data;
}

afterAll(async () => {
  if (created.length) await admin.from("customers").delete().in("id", created);
});

describe("코드 자동 채번", () => {
  it("code 없이 넣으면 CUS-숫자 형식으로 채운다", async () => {
    const row = await insertCustomer({ name: "[TEST] 코드 자동" });
    expect(row.code).toMatch(/^CUS-\d{3,}$/);
  });

  it("code 를 직접 주면 그대로 둔다", async () => {
    const row = await insertCustomer({ name: "[TEST] 코드 지정", code: "CUS-T01" });
    expect(row.code).toBe("CUS-T01");
  });

  it("자리수를 넘으면 잘리지 않고 늘어난다", async () => {
    await insertCustomer({ name: "[TEST] 999", code: "CUS-999" });
    const row = await insertCustomer({ name: "[TEST] 1000" });
    expect(row.code).toBe("CUS-1000");
  });

  it("updated_at 은 수정할 때 갱신된다", async () => {
    const row = await insertCustomer({ name: "[TEST] 수정 시각" });
    const before = await admin.from("customers").select("updated_at").eq("id", row.id).single();
    await new Promise((r) => setTimeout(r, 1100));
    await admin.from("customers").update({ memo: "변경" }).eq("id", row.id);
    const after = await admin.from("customers").select("updated_at").eq("id", row.id).single();
    expect(new Date(after.data!.updated_at).getTime()).toBeGreaterThan(
      new Date(before.data!.updated_at).getTime(),
    );
  });
});
```

참고: "자리수를 넘으면" 테스트는 운영 DB에 `CUS-1000` 이상 코드가 아직 없다는 전제다 (1단계 시점에는 데이터 없음). 테스트 후 삭제되므로 채번은 원래대로 돌아간다.

- [ ] **Step 3: 실패 확인**

Run: `pnpm test:db`
Expected: FAIL — `relation "public.customers" does not exist` (또는 `Could not find the table`)

- [ ] **Step 4: `supabase/migrations/20261015000001_schema.sql` 작성**

```sql
-- ============ enum ============
create type public.user_role as enum ('admin', 'engineer');
create type public.customer_grade as enum ('A', 'B', 'C');
create type public.inspection_cycle as enum ('monthly', 'quarterly', 'semiannual');
create type public.issue_channel as enum ('phone', 'email', 'messenger', 'inspection');
create type public.issue_category as enum ('outage', 'bug', 'question', 'config', 'feature');
create type public.issue_priority as enum ('urgent', 'high', 'normal', 'low');
create type public.issue_status as enum ('received', 'assigned', 'in_progress', 'done', 'on_hold');
create type public.issue_event_type as enum ('created', 'status_changed', 'assigned', 'comment', 'edited');
create type public.backup_status as enum ('ok', 'failed');
create type public.inspection_result as enum ('normal', 'warning', 'critical');

-- ============ 공통 트리거 함수 ============
create or replace function public.set_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at := now();
  return new;
end $$;

-- code 가 비어 있으면 "접두어-번호" 를 채운다. 인자: (접두어, 최소 자리수)
-- 기존 최대 번호 + 1, 자리수를 넘으면 늘어난다 (CUS-999 -> CUS-1000)
create or replace function public.assign_code()
returns trigger language plpgsql as $$
declare
  prefix text := tg_argv[0];
  width int := tg_argv[1]::int;
  next_num int;
begin
  if new.code is not null and new.code <> '' then
    return new;
  end if;
  perform pg_advisory_xact_lock(hashtext(tg_table_schema || '.' || tg_table_name));
  execute format(
    'select coalesce(max(substring(code from %L)::int), 0) + 1 from %I.%I',
    '^' || prefix || '-(\d+)$', tg_table_schema, tg_table_name
  ) into next_num;
  new.code := prefix || '-' || lpad(next_num::text, greatest(width, length(next_num::text)), '0');
  return new;
end $$;

-- ============ 테이블 ============
create table public.profiles (
  id uuid primary key references auth.users (id) on delete cascade,
  code text not null unique,
  name text not null,
  role public.user_role not null default 'engineer',
  team text,
  specialty text,
  email text not null,
  phone text,
  joined_on date,
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.customers (
  id uuid primary key default gen_random_uuid(),
  code text not null unique,
  name text not null,
  industry text,
  region text,
  address text,
  phone text,
  contract_type text,
  contract_start date,
  contract_end date,
  grade public.customer_grade,
  inspection_cycle public.inspection_cycle,
  primary_engineer_id uuid references public.profiles (id),
  backup_engineer_id uuid references public.profiles (id),
  memo text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.customer_contacts (
  id uuid primary key default gen_random_uuid(),
  code text not null unique,
  customer_id uuid not null references public.customers (id) on delete cascade,
  name text not null,
  department text,
  title text,
  phone text,
  email text,
  is_primary boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.products (
  id uuid primary key default gen_random_uuid(),
  name text not null unique,
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.installations (
  id uuid primary key default gen_random_uuid(),
  code text not null unique,
  customer_id uuid not null references public.customers (id) on delete cascade,
  product_id uuid not null references public.products (id),
  version text,
  installed_on date,
  last_patched_on date,
  server_name text,
  environment text,
  server_type text,
  os text,
  db text,
  was text,
  ip text,
  memo text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.issues (
  id uuid primary key default gen_random_uuid(),
  code text not null unique,
  customer_id uuid not null references public.customers (id),
  product_id uuid references public.products (id),
  product_version text,
  received_at timestamptz not null default now(),
  channel public.issue_channel not null,
  requester_contact_id uuid references public.customer_contacts (id) on delete set null,
  requester_name text,
  category public.issue_category not null,
  priority public.issue_priority not null default 'normal',
  title text not null,
  description text,
  status public.issue_status not null default 'received',
  assignee_id uuid references public.profiles (id),
  assigned_at timestamptz,
  resolved_at timestamptz,
  cause text,
  resolution text,
  kb_flag boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.issue_events (
  id uuid primary key default gen_random_uuid(),
  issue_id uuid not null references public.issues (id) on delete cascade,
  actor_id uuid references public.profiles (id) on delete set null,
  type public.issue_event_type not null,
  from_value text,
  to_value text,
  body text,
  created_at timestamptz not null default now()
);

create table public.inspections (
  id uuid primary key default gen_random_uuid(),
  code text not null unique,
  customer_id uuid not null references public.customers (id),
  installation_id uuid references public.installations (id) on delete set null,
  inspected_on date not null,
  type text,
  inspector_id uuid not null references public.profiles (id),
  cpu_pct smallint check (cpu_pct between 0 and 100),
  memory_pct smallint check (memory_pct between 0 and 100),
  disk_pct smallint check (disk_pct between 0 and 100),
  backup_status public.backup_status,
  result public.inspection_result not null,
  findings text,
  actions text,
  follow_up_issue_id uuid references public.issues (id) on delete set null,
  next_due_on date,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- ============ 인덱스 ============
create index on public.customer_contacts (customer_id);
create index on public.installations (customer_id);
create index on public.issues (customer_id);
create index on public.issues (status);
create index on public.issues (assignee_id, status);
create index on public.issue_events (issue_id, created_at);
create index on public.inspections (customer_id);
create index on public.inspections (inspected_on);

-- ============ 트리거: 코드 채번 ============
create trigger assign_code before insert on public.profiles
  for each row execute function public.assign_code('ENG', '3');
create trigger assign_code before insert on public.customers
  for each row execute function public.assign_code('CUS', '3');
create trigger assign_code before insert on public.customer_contacts
  for each row execute function public.assign_code('CON', '3');
create trigger assign_code before insert on public.installations
  for each row execute function public.assign_code('INS', '3');
create trigger assign_code before insert on public.issues
  for each row execute function public.assign_code('ISS', '4');
create trigger assign_code before insert on public.inspections
  for each row execute function public.assign_code('INSP', '4');

-- ============ 트리거: updated_at ============
create trigger set_updated_at before update on public.profiles
  for each row execute function public.set_updated_at();
create trigger set_updated_at before update on public.customers
  for each row execute function public.set_updated_at();
create trigger set_updated_at before update on public.customer_contacts
  for each row execute function public.set_updated_at();
create trigger set_updated_at before update on public.products
  for each row execute function public.set_updated_at();
create trigger set_updated_at before update on public.installations
  for each row execute function public.set_updated_at();
create trigger set_updated_at before update on public.issues
  for each row execute function public.set_updated_at();
create trigger set_updated_at before update on public.inspections
  for each row execute function public.set_updated_at();
```

- [ ] **Step 5: 운영 DB에 적용 (사용자 확인 후 실행 — 운영 DB 변경)**

```bash
pnpm db:push
```

Expected: `Applying migration 20261015000001_schema.sql... Finished supabase db push.`

- [ ] **Step 6: 통과 확인**

Run: `pnpm test:db`
Expected: PASS (4 tests)

- [ ] **Step 7: TS 타입 생성**

```bash
pnpm db:types
```

확인: `src/lib/supabase/database.types.ts` 에 `customers`, `issue_status` 가 있다.

- [ ] **Step 8: Commit**

```bash
git add supabase tests/db/codes.test.ts src/lib/supabase/database.types.ts
git commit -m "feat(db): 전체 테이블 스키마, 코드 채번, updated_at 트리거"
```

---

### Task 3: 계정과 권한 (RLS)

**Files:**
- Create: `supabase/migrations/20261015000002_auth_rls.sql`, `tests/support/accounts.ts`, `tests/db/rls.test.ts`, `scripts/create-admin.ts`, `scripts/cleanup-test-data.ts`
- Modify: `src/lib/supabase/database.types.ts` (재생성)

**Interfaces:**
- Consumes: Task 2 테이블
- Produces:
  - DB 함수 `public.is_active_user(): boolean`, `public.is_admin(): boolean` (security definer, 현재 `auth.uid()` 기준)
  - `auth.users` insert 시 `profiles` 자동 생성 (이름은 `user_metadata.name`, 역할은 항상 `engineer`)
  - `tests/support/accounts.ts`:
    - `serviceClient(): SupabaseClient<Database>`
    - `createTestAccount(role: "admin" | "engineer", opts?: { active?: boolean }): Promise<TestAccount>`
    - `signIn(account: TestAccount): Promise<SupabaseClient<Database>>`
    - `deleteTestAccounts(accounts: TestAccount[]): Promise<void>`
    - `type TestAccount = { id: string; email: string; password: string; role: "admin" | "engineer" }`

- [ ] **Step 1: 테스트 계정 도우미 — `tests/support/accounts.ts`**

```ts
import { createClient, type SupabaseClient } from "@supabase/supabase-js";
import type { Database } from "../../src/lib/supabase/database.types";

export type Role = "admin" | "engineer";
export type TestAccount = { id: string; email: string; password: string; role: Role };

const url = () => process.env.NEXT_PUBLIC_SUPABASE_URL!;
const noSession = { auth: { persistSession: false, autoRefreshToken: false } };

export function serviceClient(): SupabaseClient<Database> {
  return createClient<Database>(url(), process.env.SUPABASE_SECRET_KEY!, noSession);
}

export async function createTestAccount(
  role: Role,
  opts: { active?: boolean } = {},
): Promise<TestAccount> {
  const admin = serviceClient();
  const email = `test-${role}-${crypto.randomUUID().slice(0, 8)}@example.com`;
  const password = crypto.randomUUID();
  const { data, error } = await admin.auth.admin.createUser({
    email,
    password,
    email_confirm: true,
    user_metadata: { name: `[TEST] ${role}` },
  });
  if (error) throw error;
  const id = data.user.id;
  const { error: upErr } = await admin
    .from("profiles")
    .update({ role, active: opts.active ?? true })
    .eq("id", id);
  if (upErr) throw upErr;
  return { id, email, password, role };
}

export async function signIn(account: TestAccount): Promise<SupabaseClient<Database>> {
  const client = createClient<Database>(
    url(),
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    noSession,
  );
  const { error } = await client.auth.signInWithPassword({
    email: account.email,
    password: account.password,
  });
  if (error) throw error;
  return client;
}

export async function deleteTestAccounts(accounts: TestAccount[]): Promise<void> {
  const admin = serviceClient();
  for (const a of accounts) {
    await admin.auth.admin.deleteUser(a.id);
  }
}
```

- [ ] **Step 2: 실패하는 테스트 작성 — `tests/db/rls.test.ts`**

```ts
import { createClient, type SupabaseClient } from "@supabase/supabase-js";
import { afterAll, beforeAll, describe, expect, it } from "vitest";
import type { Database } from "../../src/lib/supabase/database.types";
import {
  createTestAccount,
  deleteTestAccounts,
  serviceClient,
  signIn,
  type TestAccount,
} from "../support/accounts";

const service = serviceClient();
let adminAcc: TestAccount, engAcc: TestAccount, inactiveAcc: TestAccount;
let asAdmin: SupabaseClient<Database>,
  asEng: SupabaseClient<Database>,
  asInactive: SupabaseClient<Database>;
const customerIds: string[] = [];

beforeAll(async () => {
  adminAcc = await createTestAccount("admin");
  engAcc = await createTestAccount("engineer");
  inactiveAcc = await createTestAccount("engineer");
  asAdmin = await signIn(adminAcc);
  asEng = await signIn(engAcc);
  asInactive = await signIn(inactiveAcc);
  // 로그인 후 비활성화 → "이미 로그인된 세션" 상황 재현
  await service.from("profiles").update({ active: false }).eq("id", inactiveAcc.id);

  const { data } = await service
    .from("customers")
    .insert({ name: "[TEST] RLS 고객사" })
    .select("id")
    .single();
  customerIds.push(data!.id);
});

afterAll(async () => {
  await service.from("customers").delete().in("id", customerIds);
  await deleteTestAccounts([adminAcc, engAcc, inactiveAcc]);
});

describe("프로필 자동 생성", () => {
  it("새 계정은 engineer 역할, ENG 코드, 메타데이터 이름으로 만들어진다", async () => {
    const { data } = await service
      .from("profiles")
      .select("code, name, email")
      .eq("id", engAcc.id)
      .single();
    expect(data!.code).toMatch(/^ENG-\d{3,}$/);
    expect(data!.name).toBe("[TEST] engineer");
    expect(data!.email).toBe(engAcc.email);
  });
});

describe("고객사 (관리자만 쓰기)", () => {
  it("엔지니어는 조회할 수 있다", async () => {
    const { data } = await asEng.from("customers").select("id").in("id", customerIds);
    expect(data).toHaveLength(1);
  });

  it("엔지니어는 등록할 수 없다", async () => {
    const { error } = await asEng.from("customers").insert({ name: "[TEST] 엔지니어 등록" });
    expect(error?.code).toBe("42501");
  });

  it("관리자는 등록할 수 있다", async () => {
    const { data, error } = await asAdmin
      .from("customers")
      .insert({ name: "[TEST] 관리자 등록" })
      .select("id")
      .single();
    expect(error).toBeNull();
    customerIds.push(data!.id);
  });
});

describe("고객담당자 (엔지니어 쓰기, 관리자만 삭제)", () => {
  let contactId: string;

  it("엔지니어는 등록할 수 있다", async () => {
    const { data, error } = await asEng
      .from("customer_contacts")
      .insert({ customer_id: customerIds[0], name: "[TEST] 담당자" })
      .select("id")
      .single();
    expect(error).toBeNull();
    contactId = data!.id;
  });

  it("엔지니어가 삭제하면 아무것도 지워지지 않는다", async () => {
    await asEng.from("customer_contacts").delete().eq("id", contactId);
    const { data } = await service.from("customer_contacts").select("id").eq("id", contactId);
    expect(data).toHaveLength(1);
  });

  it("관리자는 삭제할 수 있다", async () => {
    await asAdmin.from("customer_contacts").delete().eq("id", contactId);
    const { data } = await service.from("customer_contacts").select("id").eq("id", contactId);
    expect(data).toHaveLength(0);
  });
});

describe("프로필 권한", () => {
  it("엔지니어는 자기 역할을 admin 으로 바꿀 수 없다", async () => {
    await asEng.from("profiles").update({ role: "admin" }).eq("id", engAcc.id);
    const { data } = await service.from("profiles").select("role").eq("id", engAcc.id).single();
    expect(data!.role).toBe("engineer");
  });

  it("관리자는 다른 사람의 프로필을 수정할 수 있다", async () => {
    const { error } = await asAdmin
      .from("profiles")
      .update({ team: "기술지원1팀" })
      .eq("id", engAcc.id);
    expect(error).toBeNull();
    const { data } = await service.from("profiles").select("team").eq("id", engAcc.id).single();
    expect(data!.team).toBe("기술지원1팀");
  });
});

describe("비활성 사용자와 비로그인", () => {
  it("비활성 사용자는 어떤 고객사도 볼 수 없다", async () => {
    const { data } = await asInactive.from("customers").select("id").in("id", customerIds);
    expect(data).toEqual([]);
  });

  it("비활성 사용자는 자기 프로필도 볼 수 없다", async () => {
    const { data } = await asInactive.from("profiles").select("id").eq("id", inactiveAcc.id);
    expect(data).toEqual([]);
  });

  it("로그인하지 않으면 아무것도 볼 수 없다", async () => {
    const anon = createClient<Database>(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
      { auth: { persistSession: false } },
    );
    const { data } = await anon.from("customers").select("id");
    expect(data ?? []).toEqual([]);
  });
});
```

- [ ] **Step 3: 실패 확인**

Run: `pnpm test:db tests/db/rls.test.ts`
Expected: FAIL — `beforeAll` 에서 프로필 update 가 0건이라 이후 단언 실패, 또는 "프로필 자동 생성" 에서 `data` 가 null (프로필 트리거 없음)

- [ ] **Step 4: `supabase/migrations/20261015000002_auth_rls.sql` 작성**

```sql
-- ============ 권한 함수 ============
create or replace function public.is_active_user()
returns boolean language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.profiles where id = auth.uid() and active);
$$;

create or replace function public.is_admin()
returns boolean language sql stable security definer set search_path = public as $$
  select exists (
    select 1 from public.profiles where id = auth.uid() and active and role = 'admin'
  );
$$;

-- ============ 계정 생성 시 프로필 자동 생성 ============
-- 역할은 항상 engineer. 메타데이터는 사용자가 바꿀 수 있으므로 역할 판단에 쓰지 않는다.
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, name, email)
  values (
    new.id,
    coalesce(nullif(new.raw_user_meta_data ->> 'name', ''), split_part(new.email, '@', 1)),
    new.email
  );
  return new;
end $$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();

-- ============ RLS 켜기 ============
alter table public.profiles enable row level security;
alter table public.customers enable row level security;
alter table public.customer_contacts enable row level security;
alter table public.products enable row level security;
alter table public.installations enable row level security;
alter table public.issues enable row level security;
alter table public.issue_events enable row level security;
alter table public.inspections enable row level security;

-- ============ profiles: 조회 = 활성 사용자, 수정 = 관리자 (생성은 트리거, 삭제 없음) ============
create policy profiles_select on public.profiles
  for select to authenticated using (public.is_active_user());
create policy profiles_update on public.profiles
  for update to authenticated using (public.is_admin()) with check (public.is_admin());

-- ============ customers, products: 조회 = 활성 사용자, 쓰기 = 관리자 ============
create policy customers_select on public.customers
  for select to authenticated using (public.is_active_user());
create policy customers_insert on public.customers
  for insert to authenticated with check (public.is_admin());
create policy customers_update on public.customers
  for update to authenticated using (public.is_admin()) with check (public.is_admin());
create policy customers_delete on public.customers
  for delete to authenticated using (public.is_admin());

create policy products_select on public.products
  for select to authenticated using (public.is_active_user());
create policy products_insert on public.products
  for insert to authenticated with check (public.is_admin());
create policy products_update on public.products
  for update to authenticated using (public.is_admin()) with check (public.is_admin());
create policy products_delete on public.products
  for delete to authenticated using (public.is_admin());

-- ============ 업무 테이블: 조회·등록·수정 = 활성 사용자, 삭제 = 관리자 ============
create policy contacts_select on public.customer_contacts
  for select to authenticated using (public.is_active_user());
create policy contacts_insert on public.customer_contacts
  for insert to authenticated with check (public.is_active_user());
create policy contacts_update on public.customer_contacts
  for update to authenticated using (public.is_active_user()) with check (public.is_active_user());
create policy contacts_delete on public.customer_contacts
  for delete to authenticated using (public.is_admin());

create policy installations_select on public.installations
  for select to authenticated using (public.is_active_user());
create policy installations_insert on public.installations
  for insert to authenticated with check (public.is_active_user());
create policy installations_update on public.installations
  for update to authenticated using (public.is_active_user()) with check (public.is_active_user());
create policy installations_delete on public.installations
  for delete to authenticated using (public.is_admin());

create policy issues_select on public.issues
  for select to authenticated using (public.is_active_user());
create policy issues_insert on public.issues
  for insert to authenticated with check (public.is_active_user());
create policy issues_update on public.issues
  for update to authenticated using (public.is_active_user()) with check (public.is_active_user());
create policy issues_delete on public.issues
  for delete to authenticated using (public.is_admin());

create policy inspections_select on public.inspections
  for select to authenticated using (public.is_active_user());
create policy inspections_insert on public.inspections
  for insert to authenticated with check (public.is_active_user());
create policy inspections_update on public.inspections
  for update to authenticated using (public.is_active_user()) with check (public.is_active_user());
create policy inspections_delete on public.inspections
  for delete to authenticated using (public.is_admin());

-- ============ issue_events: 조회 = 활성 사용자, 직접 등록은 본인 코멘트만, 수정·삭제 없음 ============
create policy events_select on public.issue_events
  for select to authenticated using (public.is_active_user());
create policy events_insert_comment on public.issue_events
  for insert to authenticated
  with check (public.is_active_user() and type = 'comment' and actor_id = auth.uid());
```

- [ ] **Step 5: 운영 DB에 적용 (사용자 확인 후)**

```bash
pnpm db:push
```

- [ ] **Step 6: 통과 확인**

Run: `pnpm test:db`
Expected: PASS (codes 4 + rls 12)

- [ ] **Step 7: 남은 테스트 데이터 정리 스크립트 — `scripts/cleanup-test-data.ts`**

테스트가 중간에 죽어 `afterAll` 이 못 돈 경우에 쓴다. `[TEST]` 고객사와 `test-%@example.com` 계정만 지운다.

```ts
import { serviceClient } from "../tests/support/accounts";

async function main() {
  const db = serviceClient();

  const { data: customers } = await db.from("customers").select("id").like("name", "[TEST]%");
  const customerIds = (customers ?? []).map((c) => c.id);
  if (customerIds.length) {
    await db.from("inspections").delete().in("customer_id", customerIds);
    await db.from("issues").delete().in("customer_id", customerIds);
    await db.from("customers").delete().in("id", customerIds);
  }

  let removedUsers = 0;
  for (let page = 1; ; page++) {
    const { data, error } = await db.auth.admin.listUsers({ page, perPage: 200 });
    if (error) throw error;
    const targets = data.users.filter((u) => /^test-.+@example\.com$/.test(u.email ?? ""));
    for (const u of targets) {
      await db.auth.admin.deleteUser(u.id);
      removedUsers++;
    }
    if (data.users.length < 200) break;
  }

  console.log(`테스트 고객사 ${customerIds.length}곳, 테스트 계정 ${removedUsers}개 삭제`);
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

`package.json` scripts 에 추가:

```json
"test:cleanup": "tsx --env-file=.env.local scripts/cleanup-test-data.ts"
```

확인: `pnpm test:cleanup` → `테스트 고객사 0곳, 테스트 계정 0개 삭제` (테스트가 정상 정리했으므로)

- [ ] **Step 8: 첫 관리자 생성 스크립트 — `scripts/create-admin.ts`**

비밀번호는 사용자가 직접 입력한다. Claude는 이 스크립트를 실행하지 않고 사용자에게 실행을 요청한다.

```ts
import { createInterface } from "node:readline/promises";
import { serviceClient } from "../tests/support/accounts";

async function main() {
  const rl = createInterface({ input: process.stdin, output: process.stdout });
  const email = (await rl.question("관리자 이메일: ")).trim();
  const name = (await rl.question("이름: ")).trim();
  const password = await rl.question("비밀번호(8자 이상): ");
  rl.close();
  if (password.length < 8) throw new Error("비밀번호는 8자 이상이어야 합니다.");

  const db = serviceClient();
  const { data, error } = await db.auth.admin.createUser({
    email,
    password,
    email_confirm: true,
    user_metadata: { name },
  });
  if (error) throw error;
  const { error: upErr } = await db.from("profiles").update({ role: "admin" }).eq("id", data.user.id);
  if (upErr) throw upErr;
  console.log(`관리자 계정 생성 완료: ${email}`);
}

main().catch((e) => {
  console.error(e.message ?? e);
  process.exit(1);
});
```

`package.json` scripts 에 추가:

```json
"create-admin": "tsx --env-file=.env.local scripts/create-admin.ts"
```

사용자에게 요청: 터미널에서 `pnpm create-admin` 실행.

- [ ] **Step 9: 타입 재생성 후 Commit**

```bash
pnpm db:types
git add supabase tests scripts package.json src/lib/supabase/database.types.ts
git commit -m "feat(db): 권한 함수, 프로필 자동 생성, RLS 정책, 관리자 생성 스크립트"
```

---

### Task 4: 이슈 상태 규칙과 처리 기록

**Files:**
- Create: `supabase/migrations/20261015000003_issue_rules.sql`, `tests/db/issues.test.ts`

**Interfaces:**
- Consumes: Task 3 `is_admin()`, `tests/support/accounts.ts`
- Produces: `issues` before 트리거 `issues_apply_rules` (PRD 5.2 상태 규칙 + 배정은 관리자만), after 트리거 `issues_log_events` (`created`, `status_changed`, `assigned`, `edited` 자동 기록, `actor_id = auth.uid()`). 오류 메시지(3단계 화면에서 그대로 표시):
  - `이슈 배정은 관리자만 할 수 있습니다` (errcode 42501)
  - `담당자 없이 이 상태로 둘 수 없습니다` (errcode 23514)
  - `완료 처리하려면 조치내용이 필요합니다` (errcode 23514)

- [ ] **Step 1: 실패하는 테스트 작성 — `tests/db/issues.test.ts`**

```ts
import type { SupabaseClient } from "@supabase/supabase-js";
import { afterAll, beforeAll, describe, expect, it } from "vitest";
import type { Database } from "../../src/lib/supabase/database.types";
import {
  createTestAccount,
  deleteTestAccounts,
  serviceClient,
  signIn,
  type TestAccount,
} from "../support/accounts";

const service = serviceClient();
let adminAcc: TestAccount, engAcc: TestAccount;
let asAdmin: SupabaseClient<Database>, asEng: SupabaseClient<Database>;
let customerId: string;

async function events(issueId: string) {
  const { data } = await service
    .from("issue_events")
    .select("type, actor_id, from_value, to_value")
    .eq("issue_id", issueId)
    .order("created_at");
  return data ?? [];
}

async function newIssue(client: SupabaseClient<Database>) {
  const { data, error } = await client
    .from("issues")
    .insert({ customer_id: customerId, channel: "phone", category: "outage", title: "[TEST] 접속 불가" })
    .select("*")
    .single();
  if (error) throw error;
  return data;
}

beforeAll(async () => {
  adminAcc = await createTestAccount("admin");
  engAcc = await createTestAccount("engineer");
  asAdmin = await signIn(adminAcc);
  asEng = await signIn(engAcc);
  const { data } = await service
    .from("customers")
    .insert({ name: "[TEST] 이슈 고객사" })
    .select("id")
    .single();
  customerId = data!.id;
});

afterAll(async () => {
  await service.from("issues").delete().eq("customer_id", customerId);
  await service.from("customers").delete().eq("id", customerId);
  await deleteTestAccounts([adminAcc, engAcc]);
});

describe("이슈 등록", () => {
  it("엔지니어가 담당자 없이 등록하면 접수 상태, ISS 코드, created 기록", async () => {
    const issue = await newIssue(asEng);
    expect(issue.status).toBe("received");
    expect(issue.code).toMatch(/^ISS-\d{4,}$/);
    expect(await events(issue.id)).toEqual([
      { type: "created", actor_id: engAcc.id, from_value: null, to_value: "received" },
    ]);
  });

  it("엔지니어가 담당자를 지정해 등록할 수 없다", async () => {
    const { error } = await asEng.from("issues").insert({
      customer_id: customerId,
      channel: "phone",
      category: "bug",
      title: "[TEST] 자기 배정",
      assignee_id: engAcc.id,
    });
    expect(error?.message).toContain("이슈 배정은 관리자만 할 수 있습니다");
  });
});

describe("배정과 상태 변경", () => {
  it("관리자가 배정하면 배정 상태가 되고 배정 시각과 기록이 남는다", async () => {
    const issue = await newIssue(asEng);
    const { data, error } = await asAdmin
      .from("issues")
      .update({ assignee_id: engAcc.id })
      .eq("id", issue.id)
      .select("status, assigned_at")
      .single();
    expect(error).toBeNull();
    expect(data!.status).toBe("assigned");
    expect(data!.assigned_at).not.toBeNull();
    const types = (await events(issue.id)).map((e) => e.type);
    expect(types).toEqual(expect.arrayContaining(["created", "status_changed", "assigned"]));
  });

  it("엔지니어는 담당자를 바꿀 수 없다", async () => {
    const issue = await newIssue(asEng);
    const { error } = await asEng.from("issues").update({ assignee_id: engAcc.id }).eq("id", issue.id);
    expect(error?.message).toContain("이슈 배정은 관리자만 할 수 있습니다");
  });

  it("담당자 없이 처리중으로 바꿀 수 없다", async () => {
    const issue = await newIssue(asEng);
    const { error } = await asEng.from("issues").update({ status: "in_progress" }).eq("id", issue.id);
    expect(error?.message).toContain("담당자 없이 이 상태로 둘 수 없습니다");
  });

  it("조치내용 없이 완료할 수 없고, 있으면 완료 시각이 기록된다", async () => {
    const issue = await newIssue(asEng);
    await asAdmin.from("issues").update({ assignee_id: engAcc.id }).eq("id", issue.id);

    const fail = await asEng.from("issues").update({ status: "done" }).eq("id", issue.id);
    expect(fail.error?.message).toContain("완료 처리하려면 조치내용이 필요합니다");

    const { data } = await asEng
      .from("issues")
      .update({ status: "done", resolution: "WAS 재기동" })
      .eq("id", issue.id)
      .select("resolved_at")
      .single();
    expect(data!.resolved_at).not.toBeNull();
  });

  it("완료를 되돌리면 완료 시각이 지워진다", async () => {
    const issue = await newIssue(asEng);
    await asAdmin.from("issues").update({ assignee_id: engAcc.id }).eq("id", issue.id);
    await asEng.from("issues").update({ status: "done", resolution: "조치" }).eq("id", issue.id);
    const { data } = await asEng
      .from("issues")
      .update({ status: "in_progress" })
      .eq("id", issue.id)
      .select("resolved_at")
      .single();
    expect(data!.resolved_at).toBeNull();
  });

  it("내용을 고치면 edited 기록이 남는다", async () => {
    const issue = await newIssue(asEng);
    await asEng.from("issues").update({ priority: "urgent" }).eq("id", issue.id);
    const types = (await events(issue.id)).map((e) => e.type);
    expect(types).toContain("edited");
  });
});

describe("가져오기(서비스 키) 경로", () => {
  it("과거 시각을 직접 주면 그대로 저장된다", async () => {
    const { data, error } = await service
      .from("issues")
      .insert({
        customer_id: customerId,
        channel: "email",
        category: "outage",
        title: "[TEST] 과거 이슈",
        status: "done",
        assignee_id: engAcc.id,
        received_at: "2025-11-01T01:00:00Z",
        assigned_at: "2025-11-01T02:00:00Z",
        resolved_at: "2025-11-01T05:30:00Z",
        resolution: "디스크 정리",
      })
      .select("assigned_at, resolved_at")
      .single();
    expect(error).toBeNull();
    expect(new Date(data!.assigned_at!).toISOString()).toBe("2025-11-01T02:00:00.000Z");
    expect(new Date(data!.resolved_at!).toISOString()).toBe("2025-11-01T05:30:00.000Z");
  });
});

describe("처리 기록 권한", () => {
  it("엔지니어는 본인 이름으로 코멘트를 남길 수 있다", async () => {
    const issue = await newIssue(asEng);
    const { error } = await asEng
      .from("issue_events")
      .insert({ issue_id: issue.id, actor_id: engAcc.id, type: "comment", body: "고객 통화 완료" });
    expect(error).toBeNull();
  });

  it("코멘트가 아닌 기록은 직접 넣을 수 없다", async () => {
    const issue = await newIssue(asEng);
    const { error } = await asEng
      .from("issue_events")
      .insert({ issue_id: issue.id, actor_id: engAcc.id, type: "status_changed", to_value: "done" });
    expect(error?.code).toBe("42501");
  });

  it("기록은 관리자도 지울 수 없다", async () => {
    const issue = await newIssue(asEng);
    await asAdmin.from("issue_events").delete().eq("issue_id", issue.id);
    expect(await events(issue.id)).toHaveLength(1);
  });
});
```

- [ ] **Step 2: 실패 확인**

Run: `pnpm test:db tests/db/issues.test.ts`
Expected: FAIL — "created 기록" 단언에서 `[]` (트리거 없음), 배정 거부 단언에서 `error` 가 undefined

- [ ] **Step 3: `supabase/migrations/20261015000003_issue_rules.sql` 작성**

```sql
-- ============ 이슈 상태 규칙 (before insert/update) ============
create or replace function public.issues_apply_rules()
returns trigger language plpgsql set search_path = public as $$
begin
  -- 배정은 관리자만. 서비스 키(엑셀 가져오기 등)는 auth.uid() 가 null 이라 제외
  if auth.uid() is not null and not public.is_admin() then
    if (tg_op = 'INSERT' and new.assignee_id is not null)
       or (tg_op = 'UPDATE' and new.assignee_id is distinct from old.assignee_id) then
      raise exception '이슈 배정은 관리자만 할 수 있습니다' using errcode = '42501';
    end if;
  end if;

  if new.assignee_id is null then
    if new.status in ('assigned', 'in_progress', 'done') then
      raise exception '담당자 없이 이 상태로 둘 수 없습니다' using errcode = '23514';
    end if;
    new.assigned_at := null;
  else
    if new.status = 'received' then
      new.status := 'assigned';
    end if;
    -- 시각을 직접 주지 않았으면 지금으로 기록 (가져오기는 과거 시각을 직접 줌)
    if new.assigned_at is null
       or (tg_op = 'UPDATE'
           and new.assignee_id is distinct from old.assignee_id
           and new.assigned_at is not distinct from old.assigned_at) then
      new.assigned_at := now();
    end if;
  end if;

  if new.status = 'done' then
    if coalesce(btrim(new.resolution), '') = '' then
      raise exception '완료 처리하려면 조치내용이 필요합니다' using errcode = '23514';
    end if;
    if new.resolved_at is null then
      new.resolved_at := now();
    end if;
  else
    new.resolved_at := null;
  end if;

  return new;
end $$;

create trigger issues_apply_rules
  before insert or update on public.issues
  for each row execute function public.issues_apply_rules();

-- ============ 처리 기록 자동 생성 (after insert/update) ============
-- security definer: issue_events 는 사용자가 코멘트 외에는 직접 넣을 수 없으므로 RLS 를 우회해 기록
create or replace function public.issues_log_events()
returns trigger language plpgsql security definer set search_path = public as $$
declare
  actor uuid := auth.uid();
begin
  if tg_op = 'INSERT' then
    insert into public.issue_events (issue_id, actor_id, type, to_value)
    values (new.id, actor, 'created', new.status::text);
    return new;
  end if;

  if new.status is distinct from old.status then
    insert into public.issue_events (issue_id, actor_id, type, from_value, to_value)
    values (new.id, actor, 'status_changed', old.status::text, new.status::text);
  end if;

  if new.assignee_id is distinct from old.assignee_id then
    insert into public.issue_events (issue_id, actor_id, type, from_value, to_value)
    values (new.id, actor, 'assigned', old.assignee_id::text, new.assignee_id::text);
  end if;

  if (new.title, new.description, new.category, new.priority, new.channel,
      new.cause, new.resolution, new.product_id, new.product_version)
     is distinct from
     (old.title, old.description, old.category, old.priority, old.channel,
      old.cause, old.resolution, old.product_id, old.product_version) then
    insert into public.issue_events (issue_id, actor_id, type)
    values (new.id, actor, 'edited');
  end if;

  return new;
end $$;

create trigger issues_log_events
  after insert or update on public.issues
  for each row execute function public.issues_log_events();
```

- [ ] **Step 4: 운영 DB에 적용 (사용자 확인 후)**

```bash
pnpm db:push
```

- [ ] **Step 5: 통과 확인**

Run: `pnpm test:db`
Expected: PASS (codes 4 + rls 12 + issues 12)

- [ ] **Step 6: Commit**

```bash
git add supabase tests/db/issues.test.ts
git commit -m "feat(db): 이슈 상태 규칙과 처리 기록 트리거"
```

---

### Task 5: 로그인과 세션

**Files:**
- Create: `src/lib/supabase/server.ts`, `src/lib/supabase/proxy.ts`, `src/lib/supabase/admin.ts`, `src/proxy.ts`, `src/lib/action-state.ts`, `src/lib/auth/paths.ts`, `src/lib/validation/user.ts`, `src/app/login/{page.tsx,login-form.tsx,actions.ts}`, `src/app/forgot-password/{page.tsx,forgot-form.tsx,actions.ts}`, `src/app/set-password/{page.tsx,set-password-form.tsx,actions.ts}`, `src/app/auth/confirm/route.ts`, `src/app/auth/signout/route.ts`
- Test: `src/lib/auth/paths.test.ts`, `src/lib/validation/user.test.ts`

**Interfaces:**
- Consumes: `Database` 타입
- Produces:
  - `createClient(): Promise<SupabaseClient<Database>>` (`@/lib/supabase/server`, 사용자 세션)
  - `createAdminClient(): SupabaseClient<Database>` (`@/lib/supabase/admin`, 서비스 키, server-only)
  - `type ActionState = { error?: string; ok?: string }` (`@/lib/action-state`)
  - `isPublicPath(path: string): boolean`, `safeNextPath(next: string | null): string` (`@/lib/auth/paths`)
  - `passwordSchema` (zod, `{ password, confirm }`) (`@/lib/validation/user`)
  - 경로: `/login`, `/forgot-password`, `/set-password`, `/auth/confirm?token_hash&type&next`, `/auth/signout?reason=inactive`

- [ ] **Step 1: 실패하는 테스트 작성 — `src/lib/auth/paths.test.ts`**

```ts
import { describe, expect, it } from "vitest";
import { isPublicPath, safeNextPath } from "./paths";

describe("isPublicPath", () => {
  it.each(["/login", "/forgot-password", "/auth/confirm", "/auth/signout"])(
    "%s 는 로그인 없이 열린다",
    (p) => expect(isPublicPath(p)).toBe(true),
  );

  it.each(["/", "/admin/users", "/set-password", "/loginx"])(
    "%s 는 로그인이 필요하다",
    (p) => expect(isPublicPath(p)).toBe(false),
  );
});

describe("safeNextPath", () => {
  it("내부 경로는 그대로 쓴다", () => {
    expect(safeNextPath("/set-password")).toBe("/set-password");
  });

  it.each([null, "", "https://evil.com", "//evil.com", "/\\evil.com", "set-password"])(
    "%s 는 / 로 바꾼다",
    (n) => expect(safeNextPath(n)).toBe("/"),
  );
});
```

- [ ] **Step 2: 실패하는 테스트 작성 — `src/lib/validation/user.test.ts`**

```ts
import { describe, expect, it } from "vitest";
import { passwordSchema } from "./user";

describe("passwordSchema", () => {
  it("8자 이상이고 확인이 같으면 통과", () => {
    expect(passwordSchema.safeParse({ password: "abcd1234", confirm: "abcd1234" }).success).toBe(true);
  });

  it("8자 미만이면 안내 메시지", () => {
    const r = passwordSchema.safeParse({ password: "abc", confirm: "abc" });
    expect(r.success).toBe(false);
    expect(r.error!.issues[0].message).toBe("비밀번호는 8자 이상이어야 합니다.");
  });

  it("확인이 다르면 안내 메시지", () => {
    const r = passwordSchema.safeParse({ password: "abcd1234", confirm: "abcd12345" });
    expect(r.error!.issues[0].message).toBe("비밀번호 확인이 일치하지 않습니다.");
  });
});
```

- [ ] **Step 3: 실패 확인**

Run: `pnpm test`
Expected: FAIL — `Failed to resolve import "./paths"`, `"./user"`

- [ ] **Step 4: `src/lib/auth/paths.ts` 구현**

```ts
const PUBLIC_PATHS = ["/login", "/forgot-password"];
const PUBLIC_PREFIXES = ["/auth/"];

export function isPublicPath(path: string): boolean {
  return PUBLIC_PATHS.includes(path) || PUBLIC_PREFIXES.some((p) => path.startsWith(p));
}

// 메일 링크의 next 는 외부에서 조작될 수 있으므로 같은 사이트 내부 경로만 허용
export function safeNextPath(next: string | null): string {
  if (!next || !next.startsWith("/") || next.startsWith("//") || next.startsWith("/\\")) {
    return "/";
  }
  return next;
}
```

- [ ] **Step 5: `src/lib/validation/user.ts` 구현 (Task 6에서 내용 추가)**

```ts
import { z } from "zod";

export const passwordSchema = z
  .object({
    password: z.string().min(8, "비밀번호는 8자 이상이어야 합니다."),
    confirm: z.string(),
  })
  .refine((v) => v.password === v.confirm, {
    message: "비밀번호 확인이 일치하지 않습니다.",
    path: ["confirm"],
  });
```

- [ ] **Step 6: 통과 확인**

Run: `pnpm test`
Expected: PASS

- [ ] **Step 7: Supabase 클라이언트 3종**

`src/lib/supabase/server.ts`:

```ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import type { Database } from "./database.types";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options),
            );
          } catch {
            // 서버 컴포넌트에서는 쿠키를 쓸 수 없다. 세션 갱신은 proxy 가 처리한다.
          }
        },
      },
    },
  );
}
```

`src/lib/supabase/admin.ts`:

```ts
import "server-only";
import { createClient } from "@supabase/supabase-js";
import type { Database } from "./database.types";

export function createAdminClient() {
  return createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SECRET_KEY!,
    { auth: { persistSession: false, autoRefreshToken: false } },
  );
}
```

`src/lib/supabase/proxy.ts`:

```ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";
import { isPublicPath } from "@/lib/auth/paths";

export async function updateSession(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          response = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options),
          );
        },
      },
    },
  );

  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user && !isPublicPath(request.nextUrl.pathname)) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    url.search = "";
    return NextResponse.redirect(url);
  }

  return response;
}
```

- [ ] **Step 8: `src/proxy.ts` (Next.js 16 에서 middleware 의 새 이름)**

```ts
import type { NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/proxy";

export async function proxy(request: NextRequest) {
  return updateSession(request);
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

설치된 Next.js 가 15 이하라면 파일명을 `src/middleware.ts`, 함수명을 `middleware` 로 한다 (`pnpm list next` 로 확인).

- [ ] **Step 9: `src/lib/action-state.ts`**

```ts
export type ActionState = { error?: string; ok?: string };
```

- [ ] **Step 10: 로그인 — `src/app/login/actions.ts`**

```ts
"use server";

import { redirect } from "next/navigation";
import type { ActionState } from "@/lib/action-state";
import { createClient } from "@/lib/supabase/server";

export async function signIn(_prev: ActionState, formData: FormData): Promise<ActionState> {
  const email = String(formData.get("email") ?? "").trim();
  const password = String(formData.get("password") ?? "");
  if (!email || !password) return { error: "이메일과 비밀번호를 입력하세요." };

  const supabase = await createClient();
  const { error } = await supabase.auth.signInWithPassword({ email, password });
  if (error) return { error: "이메일 또는 비밀번호가 올바르지 않습니다." };

  redirect("/");
}
```

`src/app/login/login-form.tsx`:

```tsx
"use client";

import Link from "next/link";
import { useActionState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { signIn } from "./actions";

export function LoginForm({ notice }: { notice?: string }) {
  const [state, action, pending] = useActionState(signIn, {});

  return (
    <form action={action} className="space-y-4">
      {notice && <p className="rounded-md bg-amber-50 p-3 text-sm text-amber-800">{notice}</p>}
      <div className="space-y-2">
        <Label htmlFor="email">이메일</Label>
        <Input id="email" name="email" type="email" autoComplete="email" required />
      </div>
      <div className="space-y-2">
        <Label htmlFor="password">비밀번호</Label>
        <Input id="password" name="password" type="password" autoComplete="current-password" required />
      </div>
      {state.error && (
        <p role="alert" className="text-sm text-red-600">
          {state.error}
        </p>
      )}
      <Button type="submit" className="w-full" disabled={pending}>
        {pending ? "로그인 중..." : "로그인"}
      </Button>
      <p className="text-center text-sm">
        <Link href="/forgot-password" className="text-muted-foreground underline">
          비밀번호를 잊으셨나요?
        </Link>
      </p>
    </form>
  );
}
```

`src/app/login/page.tsx`:

```tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { LoginForm } from "./login-form";

const NOTICES: Record<string, string> = {
  inactive: "비활성화된 계정입니다. 관리자에게 문의하세요.",
  link: "링크가 만료되었거나 올바르지 않습니다. 다시 요청해 주세요.",
};

export default async function LoginPage({
  searchParams,
}: {
  searchParams: Promise<{ error?: string }>;
}) {
  const { error } = await searchParams;
  return (
    <main className="flex min-h-screen items-center justify-center bg-muted/40 p-4">
      <Card className="w-full max-w-sm">
        <CardHeader>
          <CardTitle>기술지원 통합관리</CardTitle>
        </CardHeader>
        <CardContent>
          <LoginForm notice={error ? NOTICES[error] : undefined} />
        </CardContent>
      </Card>
    </main>
  );
}
```

- [ ] **Step 11: 비밀번호 재설정 요청 — `src/app/forgot-password/actions.ts`**

```ts
"use server";

import type { ActionState } from "@/lib/action-state";
import { createClient } from "@/lib/supabase/server";

export async function requestReset(_prev: ActionState, formData: FormData): Promise<ActionState> {
  const email = String(formData.get("email") ?? "").trim();
  if (!email) return { error: "이메일을 입력하세요." };

  const supabase = await createClient();
  await supabase.auth.resetPasswordForEmail(email);
  // 계정 존재 여부를 알려주지 않기 위해 결과와 무관하게 같은 안내
  return { ok: "가입된 이메일이면 재설정 링크를 보냈습니다. 메일함을 확인하세요." };
}
```

`src/app/forgot-password/forgot-form.tsx`:

```tsx
"use client";

import Link from "next/link";
import { useActionState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { requestReset } from "./actions";

export function ForgotForm() {
  const [state, action, pending] = useActionState(requestReset, {});

  return (
    <form action={action} className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="email">이메일</Label>
        <Input id="email" name="email" type="email" autoComplete="email" required />
      </div>
      {state.error && <p role="alert" className="text-sm text-red-600">{state.error}</p>}
      {state.ok && <p className="text-sm text-green-700">{state.ok}</p>}
      <Button type="submit" className="w-full" disabled={pending}>
        재설정 링크 받기
      </Button>
      <p className="text-center text-sm">
        <Link href="/login" className="text-muted-foreground underline">로그인으로 돌아가기</Link>
      </p>
    </form>
  );
}
```

`src/app/forgot-password/page.tsx`:

```tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { ForgotForm } from "./forgot-form";

export default function ForgotPasswordPage() {
  return (
    <main className="flex min-h-screen items-center justify-center bg-muted/40 p-4">
      <Card className="w-full max-w-sm">
        <CardHeader>
          <CardTitle>비밀번호 재설정</CardTitle>
        </CardHeader>
        <CardContent>
          <ForgotForm />
        </CardContent>
      </Card>
    </main>
  );
}
```

- [ ] **Step 12: 메일 링크 확인 — `src/app/auth/confirm/route.ts`**

```ts
import type { EmailOtpType } from "@supabase/supabase-js";
import { NextResponse, type NextRequest } from "next/server";
import { safeNextPath } from "@/lib/auth/paths";
import { createClient } from "@/lib/supabase/server";

export async function GET(request: NextRequest) {
  const { searchParams, origin } = request.nextUrl;
  const tokenHash = searchParams.get("token_hash");
  const type = searchParams.get("type") as EmailOtpType | null;
  const next = safeNextPath(searchParams.get("next"));

  if (tokenHash && type) {
    const supabase = await createClient();
    const { error } = await supabase.auth.verifyOtp({ type, token_hash: tokenHash });
    if (!error) return NextResponse.redirect(new URL(next, origin));
  }

  return NextResponse.redirect(new URL("/login?error=link", origin));
}
```

- [ ] **Step 13: 새 비밀번호 설정 (초대 수락·재설정 공용) — `src/app/set-password/actions.ts`**

```ts
"use server";

import { redirect } from "next/navigation";
import type { ActionState } from "@/lib/action-state";
import { createClient } from "@/lib/supabase/server";
import { passwordSchema } from "@/lib/validation/user";

export async function setPassword(_prev: ActionState, formData: FormData): Promise<ActionState> {
  const parsed = passwordSchema.safeParse({
    password: formData.get("password"),
    confirm: formData.get("confirm"),
  });
  if (!parsed.success) return { error: parsed.error.issues[0].message };

  const supabase = await createClient();
  const { error } = await supabase.auth.updateUser({ password: parsed.data.password });
  if (error) return { error: "비밀번호를 바꾸지 못했습니다. 링크를 다시 요청해 주세요." };

  redirect("/");
}
```

`src/app/set-password/set-password-form.tsx`:

```tsx
"use client";

import { useActionState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { setPassword } from "./actions";

export function SetPasswordForm() {
  const [state, action, pending] = useActionState(setPassword, {});

  return (
    <form action={action} className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="password">새 비밀번호 (8자 이상)</Label>
        <Input id="password" name="password" type="password" autoComplete="new-password" required />
      </div>
      <div className="space-y-2">
        <Label htmlFor="confirm">비밀번호 확인</Label>
        <Input id="confirm" name="confirm" type="password" autoComplete="new-password" required />
      </div>
      {state.error && <p role="alert" className="text-sm text-red-600">{state.error}</p>}
      <Button type="submit" className="w-full" disabled={pending}>
        저장
      </Button>
    </form>
  );
}
```

`src/app/set-password/page.tsx`:

```tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { SetPasswordForm } from "./set-password-form";

export default function SetPasswordPage() {
  return (
    <main className="flex min-h-screen items-center justify-center bg-muted/40 p-4">
      <Card className="w-full max-w-sm">
        <CardHeader>
          <CardTitle>비밀번호 설정</CardTitle>
        </CardHeader>
        <CardContent>
          <SetPasswordForm />
        </CardContent>
      </Card>
    </main>
  );
}
```

- [ ] **Step 14: 로그아웃 — `src/app/auth/signout/route.ts`**

버튼은 POST, 비활성 계정 자동 로그아웃은 GET 리다이렉트로 들어온다. GET 경로는 `<Link>` 로 걸지 않는다 (미리 불러오기로 로그아웃되는 것 방지).

```ts
import { NextResponse, type NextRequest } from "next/server";
import { createClient } from "@/lib/supabase/server";

async function signOut(request: NextRequest) {
  const supabase = await createClient();
  await supabase.auth.signOut();
  const reason = request.nextUrl.searchParams.get("reason");
  const target = reason === "inactive" ? "/login?error=inactive" : "/login";
  return NextResponse.redirect(new URL(target, request.nextUrl.origin), { status: 303 });
}

export const GET = signOut;
export const POST = signOut;
```

- [ ] **Step 15: Supabase 대시보드 설정 (사용자가 직접 — 운영 설정 변경)**

1. Authentication > Sign In / Providers > **Allow new users to sign up: 끔**
2. Authentication > URL Configuration > Site URL: `http://localhost:3000` (Task 8 에서 Vercel 주소로 변경), Redirect URLs 에 `http://localhost:3000/**` 추가
3. Authentication > Emails > Templates
   - **Invite user** 링크: `{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=invite&next=/set-password`
   - **Reset Password** 링크: `{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=recovery&next=/set-password`

- [ ] **Step 16: 수동 확인**

Run: `pnpm dev` → `http://localhost:3000/login`
- 틀린 비밀번호 → "이메일 또는 비밀번호가 올바르지 않습니다."
- `pnpm create-admin` 으로 만든 계정으로 로그인 → `/` 로 이동 (아직 페이지가 없어 404 — Task 6에서 해결)
- `http://localhost:3000/auth/confirm` (토큰 없이) → `/login?error=link` 안내 표시

- [ ] **Step 17: Commit**

```bash
git add src
git commit -m "feat(auth): 로그인, 비밀번호 재설정, 메일 링크 확인, 세션 proxy"
```

---

### Task 6: 앱 레이아웃과 사용자 관리

**Files:**
- Create: `src/lib/nav.ts`, `src/lib/auth/profile.ts`, `src/components/app-sidebar.tsx`, `src/app/(app)/layout.tsx`, `src/app/(app)/page.tsx`, `src/app/(app)/admin/users/{page.tsx,invite-form.tsx,user-row.tsx,actions.ts}`
- Modify: `src/lib/validation/user.ts`
- Test: `src/lib/nav.test.ts`, `src/lib/validation/user.test.ts`

**Interfaces:**
- Consumes: `createClient`, `createAdminClient`, `ActionState`, `ROLE`, `labelOf`
- Produces:
  - `type Profile = Database["public"]["Tables"]["profiles"]["Row"]`, `requireProfile(): Promise<Profile>`, `requireAdmin(): Promise<Profile>` (`@/lib/auth/profile`)
  - `navItemsFor(role: "admin" | "engineer"): NavItem[]`, `type NavItem = { href: string; label: string; adminOnly?: boolean }`, `NAV_ITEMS` (`@/lib/nav`) — 2~4단계에서 항목 추가
  - `inviteSchema`, `inviteErrorMessage(raw: string): string`, `selfChangeError(actorId: string, targetId: string): string | null` (`@/lib/validation/user`)

- [ ] **Step 1: 실패하는 테스트 작성 — `src/lib/nav.test.ts`**

```ts
import { describe, expect, it } from "vitest";
import { navItemsFor } from "./nav";

describe("navItemsFor", () => {
  it("관리자는 사용자 관리 메뉴를 본다", () => {
    expect(navItemsFor("admin").map((i) => i.href)).toEqual(["/", "/admin/users"]);
  });

  it("엔지니어는 사용자 관리 메뉴를 보지 않는다", () => {
    expect(navItemsFor("engineer").map((i) => i.href)).toEqual(["/"]);
  });
});
```

- [ ] **Step 2: 실패하는 테스트 추가 — `src/lib/validation/user.test.ts` 아래에 이어서**

```ts
import { inviteErrorMessage, inviteSchema, selfChangeError } from "./user";

describe("inviteSchema", () => {
  it("이메일·이름·역할이 있으면 통과", () => {
    const r = inviteSchema.safeParse({ email: "a@example.com", name: "홍길동", role: "engineer" });
    expect(r.success).toBe(true);
  });

  it("이메일 형식이 틀리면 안내", () => {
    const r = inviteSchema.safeParse({ email: "abc", name: "홍길동", role: "engineer" });
    expect(r.error!.issues[0].message).toBe("올바른 이메일을 입력하세요.");
  });

  it("이름이 비면 안내", () => {
    const r = inviteSchema.safeParse({ email: "a@example.com", name: " ", role: "engineer" });
    expect(r.error!.issues[0].message).toBe("이름을 입력하세요.");
  });

  it("정의되지 않은 역할은 거부", () => {
    const r = inviteSchema.safeParse({ email: "a@example.com", name: "홍길동", role: "owner" });
    expect(r.success).toBe(false);
  });
});

describe("inviteErrorMessage", () => {
  it("이미 가입된 이메일은 한국어로 안내", () => {
    expect(inviteErrorMessage("A user with this email address has already been registered")).toBe(
      "이미 등록된 이메일입니다.",
    );
  });

  it("그 외 오류는 일반 안내", () => {
    expect(inviteErrorMessage("Email rate limit exceeded")).toBe(
      "초대 메일을 보내지 못했습니다. 잠시 후 다시 시도하세요.",
    );
  });
});

describe("selfChangeError", () => {
  it("자기 자신의 역할·활성 상태는 바꿀 수 없다", () => {
    expect(selfChangeError("u1", "u1")).toBe("자기 자신의 역할이나 활성 상태는 바꿀 수 없습니다.");
  });

  it("다른 사람은 바꿀 수 있다", () => {
    expect(selfChangeError("u1", "u2")).toBeNull();
  });
});
```

(파일 맨 위 기존 import 줄을 `import { inviteErrorMessage, inviteSchema, passwordSchema, selfChangeError } from "./user";` 하나로 합친다.)

- [ ] **Step 3: 실패 확인**

Run: `pnpm test`
Expected: FAIL — `Failed to resolve import "./nav"`, `inviteSchema` undefined

- [ ] **Step 4: `src/lib/nav.ts` 구현**

```ts
import type { Role } from "@/lib/auth/profile";

export type NavItem = { href: string; label: string; adminOnly?: boolean };

// 2~4단계에서 고객사·이슈·점검 메뉴를 추가한다
export const NAV_ITEMS: NavItem[] = [
  { href: "/", label: "홈" },
  { href: "/admin/users", label: "사용자 관리", adminOnly: true },
];

export function navItemsFor(role: Role): NavItem[] {
  return NAV_ITEMS.filter((item) => !item.adminOnly || role === "admin");
}
```

- [ ] **Step 5: `src/lib/validation/user.ts` 에 추가**

```ts
export const inviteSchema = z.object({
  email: z.string().trim().email("올바른 이메일을 입력하세요."),
  name: z.string().trim().min(1, "이름을 입력하세요."),
  role: z.enum(["admin", "engineer"]),
});

export function inviteErrorMessage(raw: string): string {
  if (/already been registered|already registered|already exists/i.test(raw)) {
    return "이미 등록된 이메일입니다.";
  }
  return "초대 메일을 보내지 못했습니다. 잠시 후 다시 시도하세요.";
}

// 관리자가 자기 자신을 강등·비활성화해 관리자가 없어지는 것을 막는다
export function selfChangeError(actorId: string, targetId: string): string | null {
  return actorId === targetId ? "자기 자신의 역할이나 활성 상태는 바꿀 수 없습니다." : null;
}
```

- [ ] **Step 6: `src/lib/auth/profile.ts` 구현**

`nav.ts` 가 `Role` 타입을 여기서 가져오므로 테스트 전에 만든다. (`server-only` 는 타입 import 에는 영향 없음)

```ts
import "server-only";
import { redirect } from "next/navigation";
import { cache } from "react";
import type { Database } from "@/lib/supabase/database.types";
import { createClient } from "@/lib/supabase/server";

export type Profile = Database["public"]["Tables"]["profiles"]["Row"];
export type Role = Database["public"]["Enums"]["user_role"];

const loadProfile = cache(async () => {
  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return { user: null, profile: null };
  // 비활성 사용자는 RLS 때문에 자기 프로필도 null 로 온다
  const { data: profile } = await supabase.from("profiles").select("*").eq("id", user.id).maybeSingle();
  return { user, profile };
});

export async function requireProfile(): Promise<Profile> {
  const { user, profile } = await loadProfile();
  if (!user) redirect("/login");
  if (!profile) redirect("/auth/signout?reason=inactive");
  return profile;
}

export async function requireAdmin(): Promise<Profile> {
  const profile = await requireProfile();
  if (profile.role !== "admin") redirect("/");
  return profile;
}
```

주의: `nav.test.ts` 는 `Role` 타입만 쓰므로 `import type` 으로 지워져 `server-only` 가 테스트에서 실행되지 않는다. `nav.ts` 의 import 가 반드시 `import type` 인지 확인.

- [ ] **Step 7: 통과 확인**

Run: `pnpm test`
Expected: PASS

- [ ] **Step 8: 사이드바 — `src/components/app-sidebar.tsx`**

```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { Button } from "@/components/ui/button";
import type { NavItem } from "@/lib/nav";
import { cn } from "@/lib/utils";

export function AppSidebar({
  items,
  name,
  roleLabel,
}: {
  items: NavItem[];
  name: string;
  roleLabel: string;
}) {
  const pathname = usePathname();

  return (
    <aside className="flex flex-col border-b bg-muted/40 md:min-h-screen md:w-56 md:border-r md:border-b-0">
      <div className="p-4 font-semibold">기술지원 통합관리</div>
      <nav className="flex gap-1 overflow-x-auto px-2 md:flex-1 md:flex-col">
        {items.map((item) => {
          const active = item.href === "/" ? pathname === "/" : pathname.startsWith(item.href);
          return (
            <Link
              key={item.href}
              href={item.href}
              className={cn(
                "rounded-md px-3 py-2 text-sm whitespace-nowrap",
                active ? "bg-primary text-primary-foreground" : "hover:bg-muted",
              )}
            >
              {item.label}
            </Link>
          );
        })}
      </nav>
      <div className="flex items-center justify-between gap-2 p-4 text-sm">
        <span>
          {name} <span className="text-muted-foreground">({roleLabel})</span>
        </span>
        <form action="/auth/signout" method="post">
          <Button type="submit" variant="ghost" size="sm">
            로그아웃
          </Button>
        </form>
      </div>
    </aside>
  );
}
```

- [ ] **Step 9: 앱 레이아웃과 홈 — `src/app/(app)/layout.tsx`, `src/app/(app)/page.tsx`**

```tsx
import { AppSidebar } from "@/components/app-sidebar";
import { requireProfile } from "@/lib/auth/profile";
import { labelOf, ROLE } from "@/lib/labels";
import { navItemsFor } from "@/lib/nav";

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const profile = await requireProfile();
  return (
    <div className="flex min-h-screen flex-col md:flex-row">
      <AppSidebar
        items={navItemsFor(profile.role)}
        name={profile.name}
        roleLabel={labelOf(ROLE, profile.role)}
      />
      <main className="flex-1 p-4 md:p-6">{children}</main>
    </div>
  );
}
```

```tsx
// src/app/(app)/page.tsx — 4단계에서 "내 업무" 화면으로 교체
import { requireProfile } from "@/lib/auth/profile";

export default async function HomePage() {
  const profile = await requireProfile();
  return <h1 className="text-xl font-semibold">안녕하세요, {profile.name}님</h1>;
}
```

- [ ] **Step 10: 사용자 관리 Server Action — `src/app/(app)/admin/users/actions.ts`**

```ts
"use server";

import { revalidatePath } from "next/cache";
import type { ActionState } from "@/lib/action-state";
import { requireAdmin } from "@/lib/auth/profile";
import { createAdminClient } from "@/lib/supabase/admin";
import { createClient } from "@/lib/supabase/server";
import { inviteErrorMessage, inviteSchema, selfChangeError } from "@/lib/validation/user";

export async function inviteUser(_prev: ActionState, formData: FormData): Promise<ActionState> {
  await requireAdmin();
  const parsed = inviteSchema.safeParse({
    email: formData.get("email"),
    name: formData.get("name"),
    role: formData.get("role"),
  });
  if (!parsed.success) return { error: parsed.error.issues[0].message };

  const admin = createAdminClient();
  const { data, error } = await admin.auth.admin.inviteUserByEmail(parsed.data.email, {
    data: { name: parsed.data.name },
  });
  if (error) return { error: inviteErrorMessage(error.message) };

  // 프로필은 트리거가 engineer 로 만든다. 관리자로 초대한 경우만 올린다.
  if (parsed.data.role === "admin") {
    const { error: roleErr } = await admin
      .from("profiles")
      .update({ role: "admin" })
      .eq("id", data.user.id);
    if (roleErr) return { error: "초대는 보냈지만 역할 지정에 실패했습니다. 목록에서 다시 지정하세요." };
  }

  revalidatePath("/admin/users");
  return { ok: `${parsed.data.email} 로 초대 메일을 보냈습니다.` };
}

export async function updateUser(_prev: ActionState, formData: FormData): Promise<ActionState> {
  const me = await requireAdmin();
  const id = String(formData.get("id") ?? "");
  const role = formData.get("role") === "admin" ? "admin" : "engineer";
  const active = formData.get("active") === "on";

  const selfError = selfChangeError(me.id, id);
  if (selfError) return { error: selfError };

  // 사용자 세션으로 수정 → RLS(profiles_update: 관리자만)가 한 번 더 확인
  const supabase = await createClient();
  const { data, error } = await supabase
    .from("profiles")
    .update({ role, active })
    .eq("id", id)
    .select("id");
  if (error || !data?.length) return { error: "변경하지 못했습니다." };

  revalidatePath("/admin/users");
  return { ok: "저장했습니다." };
}
```

- [ ] **Step 11: 초대 폼 — `src/app/(app)/admin/users/invite-form.tsx`**

```tsx
"use client";

import { useActionState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { inviteUser } from "./actions";

export function InviteForm() {
  const [state, action, pending] = useActionState(inviteUser, {});

  return (
    <form action={action} className="grid gap-3 md:grid-cols-[1fr_1fr_auto_auto] md:items-end">
      <div className="space-y-1">
        <Label htmlFor="invite-email">이메일</Label>
        <Input id="invite-email" name="email" type="email" required />
      </div>
      <div className="space-y-1">
        <Label htmlFor="invite-name">이름</Label>
        <Input id="invite-name" name="name" required />
      </div>
      <div className="space-y-1">
        <Label htmlFor="invite-role">역할</Label>
        <select id="invite-role" name="role" defaultValue="engineer" className="h-9 rounded-md border px-2 text-sm">
          <option value="engineer">엔지니어</option>
          <option value="admin">관리자</option>
        </select>
      </div>
      <Button type="submit" disabled={pending}>
        {pending ? "보내는 중..." : "초대"}
      </Button>
      {state.error && <p role="alert" className="text-sm text-red-600 md:col-span-4">{state.error}</p>}
      {state.ok && <p className="text-sm text-green-700 md:col-span-4">{state.ok}</p>}
    </form>
  );
}
```

- [ ] **Step 12: 사용자 행 — `src/app/(app)/admin/users/user-row.tsx`**

```tsx
"use client";

import { useActionState } from "react";
import { Button } from "@/components/ui/button";
import type { Profile } from "@/lib/auth/profile";
import { updateUser } from "./actions";

export function UserRow({ user, isMe }: { user: Profile; isMe: boolean }) {
  const [state, action, pending] = useActionState(updateUser, {});

  return (
    <tr className="border-b align-middle">
      <td className="p-2 font-mono text-xs">{user.code}</td>
      <td className="p-2">{user.name}</td>
      <td className="p-2 text-muted-foreground">{user.email}</td>
      <td className="p-2" colSpan={3}>
        <form action={action} className="flex flex-wrap items-center gap-3">
          <input type="hidden" name="id" value={user.id} />
          <select
            name="role"
            defaultValue={user.role}
            disabled={isMe}
            aria-label={`${user.name} 역할`}
            className="h-8 rounded-md border px-2 text-sm"
          >
            <option value="engineer">엔지니어</option>
            <option value="admin">관리자</option>
          </select>
          <label className="flex items-center gap-1 text-sm">
            <input type="checkbox" name="active" defaultChecked={user.active} disabled={isMe} />
            활성
          </label>
          {!isMe && (
            <Button type="submit" size="sm" variant="outline" disabled={pending}>
              저장
            </Button>
          )}
          {state.error && <span role="alert" className="text-sm text-red-600">{state.error}</span>}
          {state.ok && <span className="text-sm text-green-700">{state.ok}</span>}
        </form>
      </td>
    </tr>
  );
}
```

- [ ] **Step 13: 사용자 관리 화면 — `src/app/(app)/admin/users/page.tsx`**

```tsx
import { requireAdmin } from "@/lib/auth/profile";
import { createClient } from "@/lib/supabase/server";
import { InviteForm } from "./invite-form";
import { UserRow } from "./user-row";

export default async function UsersPage() {
  const me = await requireAdmin();
  const supabase = await createClient();
  const { data: users } = await supabase.from("profiles").select("*").order("code");

  return (
    <div className="space-y-6">
      <h1 className="text-xl font-semibold">사용자 관리</h1>
      <section className="space-y-2 rounded-lg border p-4">
        <h2 className="font-medium">새 사용자 초대</h2>
        <InviteForm />
      </section>
      <div className="overflow-x-auto">
        <table className="w-full text-sm">
          <thead>
            <tr className="border-b text-left text-muted-foreground">
              <th className="p-2">코드</th>
              <th className="p-2">이름</th>
              <th className="p-2">이메일</th>
              <th className="p-2" colSpan={3}>역할 / 상태</th>
            </tr>
          </thead>
          <tbody>
            {(users ?? []).map((u) => (
              <UserRow key={u.id} user={u} isMe={u.id === me.id} />
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
}
```

- [ ] **Step 14: 빌드·린트 확인**

Run: `pnpm lint && pnpm build`
Expected: 오류 없음

- [ ] **Step 15: 수동 확인**

Run: `pnpm dev`
- 관리자로 로그인 → "안녕하세요, ○○님", 사이드바에 홈·사용자 관리
- 사용자 관리에서 본인 행은 역할·활성 변경 불가(비활성화된 입력)
- 본인 이메일로 다시 초대 → "이미 등록된 이메일입니다."

- [ ] **Step 16: Commit**

```bash
git add src
git commit -m "feat: 앱 레이아웃, 역할별 메뉴, 사용자 초대·역할·활성 관리"
```

---

### Task 7: E2E — 로그인과 권한 흐름

**Files:**
- Create: `playwright.config.ts`, `tests/e2e/global-setup.ts`, `tests/e2e/global-teardown.ts`, `tests/e2e/auth.spec.ts`

**Interfaces:**
- Consumes: `tests/support/accounts.ts` (`createTestAccount`, `deleteTestAccounts`, `serviceClient`), 화면 문구 (Task 5·6)
- Produces: `pnpm test:e2e` 로 실행되는 브라우저 테스트

- [ ] **Step 1: `playwright.config.ts`**

```ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "tests/e2e",
  globalSetup: "./tests/e2e/global-setup.ts",
  globalTeardown: "./tests/e2e/global-teardown.ts",
  workers: 1,
  use: { baseURL: "http://localhost:3000" },
  webServer: {
    command: "pnpm dev",
    url: "http://localhost:3000/login",
    reuseExistingServer: true,
    timeout: 120_000,
  },
});
```

- [ ] **Step 2: 테스트 계정 준비·정리 — `tests/e2e/global-setup.ts`, `tests/e2e/global-teardown.ts`**

```ts
// tests/e2e/global-setup.ts
import { writeFileSync } from "node:fs";
import { createTestAccount } from "../support/accounts";

export default async function globalSetup() {
  process.loadEnvFile(".env.local");
  const admin = await createTestAccount("admin");
  const engineer = await createTestAccount("engineer");
  writeFileSync("tests/e2e/.users.json", JSON.stringify({ admin, engineer }));
}
```

```ts
// tests/e2e/global-teardown.ts
import { readFileSync, rmSync } from "node:fs";
import { deleteTestAccounts, type TestAccount } from "../support/accounts";

export default async function globalTeardown() {
  process.loadEnvFile(".env.local");
  const users = JSON.parse(readFileSync("tests/e2e/.users.json", "utf8")) as Record<string, TestAccount>;
  await deleteTestAccounts(Object.values(users));
  rmSync("tests/e2e/.users.json");
}
```

- [ ] **Step 3: 테스트 작성 — `tests/e2e/auth.spec.ts`**

```ts
import { readFileSync } from "node:fs";
import { expect, test, type Page } from "@playwright/test";
import { serviceClient, type TestAccount } from "../support/accounts";

process.loadEnvFile(".env.local");
// global-setup 이 만든 파일은 테스트 수집 이후에 생기므로 실행 시점에 읽는다
let users: { admin: TestAccount; engineer: TestAccount };
test.beforeAll(() => {
  users = JSON.parse(readFileSync("tests/e2e/.users.json", "utf8"));
});

async function login(page: Page, account: { email: string; password: string }) {
  await page.goto("/login");
  await page.getByLabel("이메일").fill(account.email);
  await page.getByLabel("비밀번호").fill(account.password);
  await page.getByRole("button", { name: "로그인" }).click();
}

test("로그인하지 않으면 로그인 화면으로 간다", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveURL(/\/login$/);
});

test("틀린 비밀번호는 안내 메시지", async ({ page }) => {
  await login(page, { email: users.admin.email, password: "wrong-password" });
  await expect(page.getByRole("alert")).toHaveText("이메일 또는 비밀번호가 올바르지 않습니다.");
});

test("관리자는 사용자 관리에 들어갈 수 있다", async ({ page }) => {
  await login(page, users.admin);
  await expect(page.getByRole("heading", { name: /안녕하세요/ })).toBeVisible();
  await page.getByRole("link", { name: "사용자 관리" }).click();
  await expect(page.getByRole("heading", { name: "사용자 관리" })).toBeVisible();
  await expect(page.getByText(users.engineer.email)).toBeVisible();
});

test("엔지니어는 사용자 관리 메뉴가 없고 주소로 들어가도 홈으로 간다", async ({ page }) => {
  await login(page, users.engineer);
  await expect(page.getByRole("heading", { name: /안녕하세요/ })).toBeVisible();
  await expect(page.getByRole("link", { name: "사용자 관리" })).toHaveCount(0);
  await page.goto("/admin/users");
  await expect(page).toHaveURL(/\/$/);
});

test("로그인 중에 비활성화되면 다음 이동 때 로그인 화면으로 나간다", async ({ page }) => {
  await login(page, users.engineer);
  await expect(page.getByRole("heading", { name: /안녕하세요/ })).toBeVisible();

  const db = serviceClient();
  await db.from("profiles").update({ active: false }).eq("id", users.engineer.id);
  try {
    await page.reload();
    await expect(page).toHaveURL(/\/login\?error=inactive$/);
    await expect(page.getByText("비활성화된 계정입니다.", { exact: false })).toBeVisible();
  } finally {
    await db.from("profiles").update({ active: true }).eq("id", users.engineer.id);
  }
});
```

- [ ] **Step 4: 실행**

Run: `pnpm test:e2e`
Expected: PASS (5 tests). 실패하면 `pnpm exec playwright show-report` 로 화면 확인 후 원인 수정 (문구·라벨 불일치가 가장 흔함).

- [ ] **Step 5: 전체 테스트 한 번에**

Run: `pnpm test && pnpm test:db && pnpm test:e2e && pnpm test:cleanup`
Expected: 모두 PASS, 마지막 줄 `테스트 고객사 0곳, 테스트 계정 0개 삭제`

- [ ] **Step 6: Commit**

```bash
git add playwright.config.ts tests/e2e
git commit -m "test(e2e): 로그인, 역할별 접근, 비활성 계정 흐름"
```

---

### Task 8: Vercel 배포

**Files:**
- Create: `README.md` (create-next-app 기본 README 교체)

**Interfaces:**
- Consumes: 전체
- Produces: Vercel 운영 URL

- [ ] **Step 1: `README.md` 를 아래 내용으로 교체**

````markdown
# 기술지원 통합관리 시스템

사내 기술지원 업무(고객사·이슈·점검)를 한곳에서 관리하는 웹 시스템. 요구사항은 `prd.md`.

## 로컬 실행

```bash
pnpm install
cp .env.example .env.local   # Supabase 값 입력
pnpm dev
```

## 테스트

| 명령 | 내용 |
|---|---|
| `pnpm test` | 단위 테스트 |
| `pnpm test:db` | DB 권한·규칙 테스트 (운영 Supabase에 `[TEST]` 데이터를 만들고 지움) |
| `pnpm test:e2e` | 브라우저 테스트 |
| `pnpm test:cleanup` | 테스트가 중간에 멈춰 남은 테스트 데이터 정리 |

## DB 변경

```bash
# supabase/migrations/ 에 새 SQL 파일 추가 후
pnpm db:push
pnpm db:types
```

주의: 운영 DB 하나만 사용한다. `supabase db reset` 은 절대 실행하지 않는다.

## 첫 관리자 만들기

```bash
pnpm create-admin
```
````

```bash
git add README.md
git commit -m "docs: README"
```

- [ ] **Step 2: GitHub 저장소에 올리기 (사용자 확인 후 — 외부 게시)**

사용자에게 비공개 저장소 생성 여부를 확인한 뒤:

```bash
gh repo create tech-support-hub --private --source . --push
```

- [ ] **Step 3: Vercel 프로젝트 연결 (사용자가 직접)**

1. vercel.com > Add New > Project > 위 저장소 Import
2. Environment Variables 에 `.env.local` 의 3개 값 입력 (`SUPABASE_SECRET_KEY` 는 Sensitive 체크)
3. Deploy

- [ ] **Step 4: Supabase URL 설정 갱신 (사용자가 직접)**

Authentication > URL Configuration
- Site URL: `https://<vercel 프로젝트>.vercel.app`
- Redirect URLs: `https://<vercel 프로젝트>.vercel.app/**` 추가 (`http://localhost:3000/**` 는 개발용으로 유지)

- [ ] **Step 5: 배포 확인**

배포 URL에서:
- `/` → `/login` 으로 이동
- 관리자 로그인 → 홈, 사용자 관리 열림
- 사용자 관리에서 동료 1명 초대 → 초대 메일 링크 클릭 → 비밀번호 설정 화면 → 저장 후 홈 진입

이 다섯 가지가 되면 1단계 완료.
