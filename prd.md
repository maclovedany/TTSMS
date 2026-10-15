# 기술지원 통합관리 시스템 PRD

- 작성일: 2026-10-15
- 범위: 1차 릴리스 (고객사 + 이슈·점검 기록)
- 참고 데이터: `기술지원_더미데이터.xlsx`

---

## 1. 배경과 문제

기술지원 업무 정보가 개인 엑셀·메일·메신저에 흩어져 있어 다음 문제가 반복된다.

| 문제 | 현상 |
|---|---|
| 정보 분산 | 고객사별 점검기록·이슈·담당자·버전 정보를 찾으려면 여러 파일과 사람을 거쳐야 함 |
| 담당자 의존 | 고객사 사정을 담당 엔지니어만 알고 있어, 부재·교체 시 대응이 멈춤 |
| 답변 지연 | 고객 환경(버전·서버·이력)을 파악하는 데 시간이 걸림 |
| 업무량 불균형 | 누가 몇 건을 맡고 있는지 보이지 않아 특정 인원에게 업무가 몰림 |
| 같은 문제 반복 | 과거 해결 사례가 남지 않아 같은 원인의 이슈를 매번 처음부터 조사 |

## 2. 목표

### 2.1 제품 목표 (1차)

1. 고객사 하나를 열면 기본정보·계약·담당자·설치환경·이슈 이력·점검 기록이 **한 화면**에 모인다.
2. 이슈의 접수부터 완료까지 상태와 처리 기록이 남아, **담당자가 바뀌어도 맥락이 이어진다.**
3. 관리자는 엔지니어별 미완료 업무량을 한눈에 보고 배정한다.
4. 이슈를 처리할 때 **비슷한 과거 완료 이슈**가 함께 보인다.
5. 기존 엑셀 데이터를 일괄로 가져온다.

### 2.2 성공 기준

| 기준 | 측정 방법 |
|---|---|
| 고객사 현황 파악이 1분 이내 | 고객사 검색 → 상세 화면에서 모든 탭 확인 |
| 기존 엑셀 데이터 100% 이전 | 표준 양식 엑셀 가져오기 후 시트별 건수 일치 |
| 미배정 이슈와 엔지니어별 미완료 건수가 홈에 표시 | 관리자 홈 화면 확인 |
| 이슈 상세에서 유사 이슈 추천 | 같은 제품·분류의 완료 이슈가 있으면 1건 이상 표시 |

### 2.3 1차에서 하지 않는 것

- 지식베이스(KB) 본문 작성·검색 → 2차. 1차는 이슈의 `KB등록여부` 표시만 저장
- 월별 기술지원 일정(캘린더), 자동 배정 → 2차
- 알림(메일·메신저), SLA 관리
- 고객이 직접 접속하는 포털
- 모바일 전용 화면 (반응형으로 깨지지 않는 수준만)

## 3. 사용자와 권한

- 사용 규모: 사내 사용자 약 20명 이하, 고객사 약 100곳 이하
- 회원가입 없음. 관리자가 이메일로 계정을 초대

| 역할 | 할 수 있는 일 |
|---|---|
| 엔지니어 (`engineer`) | 전체 고객사 조회 / 이슈(상태 변경 포함)·점검·설치환경·고객담당자 등록·수정 |
| 관리자 (`admin`) | 엔지니어 권한 전체 + 고객사 등록·수정·삭제, 이슈 배정, 계정 초대·역할 변경, 제품 관리, 엑셀 가져오기, 모든 데이터 삭제 |

- 삭제는 관리자만 가능
- 권한은 화면에서 숨기는 것과 별개로 **DB의 행 단위 보안(RLS)** 으로 강제

## 4. 기술 구성

| 영역 | 선택 |
|---|---|
| 프레임워크 | Next.js (App Router, TypeScript) |
| 호스팅 | Vercel |
| DB·인증·파일 | Supabase (Postgres, Auth, Storage) |
| DB 접근 | `@supabase/ssr` 클라이언트 직접 사용, ORM 없음 |
| 권한 | Supabase RLS 정책 |
| 스키마 관리 | Supabase CLI SQL 마이그레이션 |
| UI | Tailwind CSS + shadcn/ui |
| 엑셀 처리 | 서버(Route Handler / Server Action)에서 파싱 |
| 테스트 | Vitest(단위), Playwright(E2E) |

## 5. 데이터 모델

### 5.1 공통 규칙

- 내부 키는 `uuid`. 화면 표시·엑셀 연동용으로 사람이 읽는 코드(`CUS-001`, `ISS-0001` 등)를 별도 컬럼 `code`(unique)로 둔다.
- 선택값은 DB에 영문 코드로 저장하고 화면에서 한글로 표시한다.
- 모든 테이블에 `created_at`, `updated_at` 을 둔다.
- 계산 가능한 값은 저장하지 않는다: 계약상태(종료일 기준), 처리시간(배정~완료 시각 차이).

### 5.2 테이블

**`profiles`** — 사내 사용자 (엑셀 "엔지니어")

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | `auth.users.id` 참조 |
| code | text unique | `ENG-001` |
| name | text | |
| role | enum | `admin`, `engineer` |
| team | text | |
| specialty | text | 전문분야 |
| email | text | |
| phone | text | |
| joined_on | date | 입사일 |
| active | boolean | 퇴사 시 false (삭제 대신) |

**`customers`** — 고객사

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| code | text unique | `CUS-001` |
| name | text | |
| industry | text | 업종 |
| region | text | |
| address | text | |
| phone | text | 대표전화 |
| contract_type | text | 유지보수 / 구축+유지보수 / 라이선스 |
| contract_start | date | |
| contract_end | date | 계약상태는 이 값으로 계산 |
| grade | enum | `A`, `B`, `C` |
| inspection_cycle | enum | `monthly`, `quarterly`, `semiannual` |
| primary_engineer_id | uuid FK → profiles | 정담당 |
| backup_engineer_id | uuid FK → profiles | 부담당 |
| memo | text | |

**`customer_contacts`** — 고객사 측 담당자

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| code | text unique | `CON-001` |
| customer_id | uuid FK → customers | |
| name, department, title | text | 이름, 부서, 직급 |
| phone, email | text | |
| is_primary | boolean | 주담당 여부 |

**`products`** — 제품

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| name | text unique | 예: `SmartGate (접근통제)` |
| active | boolean | |

**`installations`** — 설치환경 (제품·버전 + 서버 환경)

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| code | text unique | `INS-001` |
| customer_id | uuid FK → customers | |
| product_id | uuid FK → products | |
| version | text | 제품 버전 |
| installed_on | date | |
| last_patched_on | date | nullable |
| server_name | text | |
| environment | text | 운영 / 운영(이중화) / 개발/테스트 |
| server_type | text | 물리서버 / VM / 클라우드 |
| os, db, was | text | |
| ip | text | |
| memo | text | |

**`issues`** — 이슈

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| code | text unique | `ISS-0001`, 새 이슈는 자동 채번 |
| customer_id | uuid FK → customers | |
| product_id | uuid FK → products | nullable |
| product_version | text | 접수 시점 버전 (스냅샷) |
| received_at | timestamptz | 접수일시 |
| channel | enum | `phone`, `email`, `messenger`, `inspection` |
| requester_contact_id | uuid FK → customer_contacts | nullable |
| requester_name | text | 연결이 없을 때도 표시용 |
| category | enum | `outage`(장애), `bug`(버그), `question`(문의), `config`(설정변경), `feature`(기능요청) |
| priority | enum | `urgent`, `high`, `normal`, `low` |
| title | text | |
| description | text | |
| status | enum | `received` → `assigned` → `in_progress` → `done`, 그리고 `on_hold` |
| assignee_id | uuid FK → profiles | nullable |
| assigned_at | timestamptz | nullable |
| resolved_at | timestamptz | nullable |
| cause | text | 원인 |
| resolution | text | 조치내용 |
| kb_flag | boolean | KB등록여부 (2차 KB 연동용) |

상태 규칙:

- `received` 는 담당자가 없다. 담당자를 지정하면 `assigned` 로 바뀌고 `assigned_at` 이 기록된다.
- `done` 으로 바꾸려면 `resolution` 이 필수. `resolved_at` 이 기록된다.
- `done` 에서 다른 상태로 되돌리면 `resolved_at` 을 비운다.

**`issue_events`** — 이슈 처리 기록

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| issue_id | uuid FK → issues | |
| actor_id | uuid FK → profiles | |
| type | enum | `created`, `status_changed`, `assigned`, `comment`, `edited` |
| from_value, to_value | text | 상태·담당자 변경 전후 |
| body | text | 코멘트 내용 |
| created_at | timestamptz | |

- 상태·담당자 변경 시 자동으로 기록한다.
- 수정·삭제 불가 (관리자 포함). 이력 보존이 목적.

**`inspections`** — 점검 기록

| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | uuid PK | |
| code | text unique | `INSP-0001` |
| customer_id | uuid FK → customers | |
| installation_id | uuid FK → installations | nullable, 대상 서버 |
| inspected_on | date | |
| type | text | 정기점검(방문) / 정기점검(원격) / 긴급점검 / 업그레이드 사전점검 |
| inspector_id | uuid FK → profiles | |
| cpu_pct, memory_pct, disk_pct | smallint | 0~100 |
| backup_status | enum | `ok`, `failed` |
| result | enum | `normal`(정상), `warning`(주의), `critical`(이상) |
| findings | text | 점검소견 |
| actions | text | 조치사항 |
| follow_up_issue_id | uuid FK → issues | nullable |
| next_due_on | date | 차기점검예정일 (점검일 + 고객사 점검주기로 기본값 제안) |

## 6. 화면

공통: 왼쪽 메뉴 + 본문. 목록은 검색·필터·정렬·페이지 이동을 지원한다.

### 6.1 로그인
- 이메일 + 비밀번호 (Supabase Auth). 비밀번호 재설정 메일 지원.

### 6.2 홈 (내 업무)
- 나에게 배정된 미완료 이슈 (우선순위·접수일 순)
- 이번 달 점검 예정 고객사 (내가 정·부담당인 곳)
- 관리자 추가 영역: 미배정 이슈 목록, 엔지니어별 미완료 이슈 건수

### 6.3 고객사 목록
- 검색: 고객사명, 코드
- 필터: 업종, 등급, 계약상태(계약중/만료/60일 내 만료), 담당 엔지니어
- 표시: 미완료 이슈 수, 최근 점검일, 계약 만료 임박 표시

### 6.4 고객사 상세 (핵심 화면)
- 상단: 기본정보, 계약, 정·부 담당 엔지니어
- 탭: 이슈 이력 / 점검 기록 / 설치환경 / 고객 담당자
- 이 화면에서 이슈·점검·설치환경·담당자를 바로 등록

### 6.5 이슈 목록
- 필터: 상태, 분류, 우선순위, 담당자, 고객사, 제품, 접수 기간
- 검색: 제목, 내용, 원인, 조치내용

### 6.6 이슈 상세
- 내용, 상태 변경, 담당자 배정(관리자), 원인·조치 입력
- 처리 기록 타임라인과 코멘트 (`issue_events`)
- **유사 이슈**: 아래 조건을 만족하는 `done` 이슈를 최대 5건 표시
  - 같은 제품이면 우선, 같은 분류이면 우선
  - 제목·원인 키워드가 겹치는 정도로 정렬 (Postgres 전문검색 또는 trigram 유사도)

### 6.7 점검 목록·등록
- 필터: 고객사, 점검자, 결과, 기간
- 결과가 `critical` 이면 저장 시 "후속 이슈 등록" 선택 가능 → 고객사·제품·내용을 채운 이슈를 만들고 `follow_up_issue_id` 연결

### 6.8 관리 (관리자 전용)
- 사용자: 초대, 역할 변경, 비활성화
- 제품: 추가, 이름 변경, 비활성화
- 엑셀 가져오기 (7장)

## 7. 엑셀 가져오기

### 7.1 표준 양식
- `기술지원_더미데이터.xlsx` 와 같은 시트 이름·컬럼 헤더를 표준으로 한다: 고객사, 고객담당자, 설치환경, 엔지니어, 이슈, 점검기록 (`안내` 시트는 무시)
- 관리 화면에서 빈 양식을 내려받을 수 있다.
- 일부 시트만 있는 파일도 허용한다.

### 7.2 흐름
1. 파일 업로드
2. 서버에서 파싱·검증 → 미리보기: 시트별 추가 N건 / 수정 N건 / 오류 N건, 오류는 `시트명 행번호: 이유` 로 표시
3. 오류가 0건일 때만 "가져오기" 버튼 활성화
4. 한 트랜잭션으로 저장 (중간 실패 시 전체 취소)

### 7.3 규칙
- 처리 순서: 엔지니어 → 고객사 → 고객담당자 / 설치환경 → 이슈 → 점검기록
- `code` 기준 upsert: 같은 코드는 덮어쓰고 새 코드는 추가. 같은 파일을 다시 올려도 중복이 생기지 않는다.
- "엔지니어" 시트: 해당 이메일의 계정이 없으면 초대 메일을 보내고 프로필을 만든다.
- 이슈·점검기록의 엔지니어 이름은 `profiles.name` 과 맞춘다. 없거나 동명이인이면 오류.
- 제품명이 `products` 에 없으면 새로 추가한다 (미리보기에 표시).
- 이슈의 요청자(`이름 직급`)는 같은 고객사 담당자 이름과 맞으면 `requester_contact_id` 연결, 아니면 `requester_name` 만 저장.
- 점검기록의 대상서버는 같은 고객사의 `installations.server_name` 과 맞춘다. 없으면 비워 둔다.
- 한글 선택값(예: `완료`, `장애`)은 영문 코드로 변환한다. 정의되지 않은 값이면 오류.
- 계산 컬럼(`계약상태`, `처리시간(h)`)은 읽지 않는다.

## 8. 오류 처리

- 입력 폼: 저장 전 필수값·형식·상태 규칙(5.2) 검증, 필드 옆에 오류 메시지
- 권한 없는 요청: RLS가 거부, 화면에 "권한이 없습니다" 안내
- 동시 수정: 마지막 저장 우선. 변경 이력은 `issue_events` 에 남음
- 네트워크·서버 오류: 입력 내용을 유지한 채 오류 안내, 다시 시도 가능

## 9. 비기능 요구사항

- 성능: 목록·상세 화면 2초 이내 (고객사 100곳, 이슈 수천 건 기준)
- 보안: 모든 테이블 RLS 활성화, 서비스 키는 서버에서만 사용, HTTPS (Vercel 기본)
- 브라우저: 최신 Chrome, Edge, Safari
- 백업: Supabase 기본 일일 백업

## 10. 테스트

| 종류 | 대상 |
|---|---|
| 단위 (Vitest) | 엑셀 파싱·검증·값 변환, 상태 전이 규칙, 처리시간 계산, 유사 이슈 정렬 |
| 권한 (Vitest + 로컬 Supabase) | 엔지니어·관리자 계정으로 각 테이블 읽기/쓰기/삭제 허용·거부 확인 |
| E2E (Playwright) | 로그인 → 고객사 상세 → 이슈 등록 → 배정 → 완료, 엑셀 가져오기 미리보기·저장 |
| 수동 | 개발용 Supabase에 더미 엑셀을 가져와 모든 화면 점검 |

## 11. 이후 단계 (참고)

| 단계 | 내용 |
|---|---|
| 2차 | 엔지니어 배정 보드, 월별 기술지원 일정(정기점검 자동 생성) |
| 2차 | 지식베이스: 완료 이슈를 KB 문서로 전환, 검색, 이슈 상세에서 추천 |
| 3차 | 대시보드(고객사·제품·엔지니어별 통계), 알림, SLA |
