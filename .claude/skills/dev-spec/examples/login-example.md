# Few-shot 예시: 이메일 로그인/회원가입

> 이 파일은 skill.md에서 import되어 Claude가 작업 단위 분해의 품질 기준으로 참조합니다.
> 아래는 "이메일 로그인/회원가입" 요구사항에 대한 분해 결과의 완전한 예시입니다.

---

## 원본 요구사항

> 이메일과 비밀번호로 회원가입 및 로그인할 수 있어야 한다.
> 로그인 시 JWT 토큰을 발급하고, 토큰 갱신이 가능해야 한다.
> 5회 로그인 실패 시 계정을 10분간 잠금한다.

---

## _index.md 예시

작업 단위 목록:

| ID | 제목 | 난이도 | 의존관계 |
|----|------|--------|---------|
| WU-01 | 인증 기반 설정 | S | 없음 |
| WU-02 | 사용자 테이블 & 모델 | S | 없음 |
| WU-03 | 회원가입 API | M | WU-01, WU-02 |
| WU-04 | 로그인 API | M | WU-01, WU-02 |
| WU-05 | 토큰 갱신 API | S | WU-01 |
| WU-06 | 로그인 실패 횟수 제한 | M | WU-04 |

의존관계 맵:
```
WU-01 (인증 설정)  ──┬──→ WU-03 (회원가입)
                     ├──→ WU-04 (로그인) ──→ WU-06 (실패 제한)
                     └──→ WU-05 (토큰 갱신)
WU-02 (사용자 DB)  ──┬──→ WU-03
                     └──→ WU-04
```

추천 구현 순서:
- Phase 1 (병렬): WU-01 + WU-02
- Phase 2 (병렬): WU-03 + WU-04 + WU-05
- Phase 3: WU-06

---

## WU별 상세 기획서 예시

아래는 WU-04 (로그인 API)의 상세 기획서 전체 예시입니다.
**모든 WU를 이 수준으로 작성해야 합니다.**

---

# WU-04: 로그인 API

> 난이도: M | 의존관계: WU-01 (JWT 유틸), WU-02 (users 테이블)

---

## 1. 개요

이메일+비밀번호로 사용자를 인증하고 JWT 토큰 쌍(access + refresh)을 발급하는 API.
잠금 상태인 계정의 로그인을 차단하며, 응답에서 이메일 존재 여부를 노출하지 않는다.

---

## 2. API 스펙

### POST /auth/login

**Request:**
- Headers: Content-Type: application/json
- Body:
```json
{
  "email": "user@example.com",
  "password": "securePass123"
}
```
- Validation Rules:
  - email: required, 유효한 이메일 형식
  - password: required, 빈 문자열 불가

**Response:**

| Status | 조건 | Body |
|--------|------|------|
| 200 | 인증 성공 | `{ "accessToken": "eyJ...", "refreshToken": "eyJ..." }` |
| 400 | 입력값 누락/형식 오류 | `{ "error": "email은 필수입니다" }` |
| 401 | 이메일 없음 또는 비밀번호 불일치 | `{ "error": "이메일 또는 비밀번호가 올바르지 않습니다" }` |
| 423 | 계정 잠금 상태 | `{ "error": "계정이 잠금되었습니다", "lockedUntil": "2026-04-09T10:30:00Z" }` |

---

## 3. 데이터 모델

### 테이블: users (WU-02에서 생성됨 — 이 WU에서는 조회만)

| 컬럼 | 사용 방식 |
|------|----------|
| email | WHERE 조건으로 사용자 조회 |
| password_hash | bcrypt.compare로 비밀번호 검증 |
| locked_until | 현재 시간과 비교하여 잠금 여부 판단 |

---

## 4. 비즈니스 로직

### 처리 흐름
1. request body에서 email, password 추출 및 유효성 검증
2. email 소문자 정규화 후 users 테이블 조회
   - 사용자 없음 → 401 반환 (이메일 없다고 알려주지 않음)
3. locked_until 확인
   - locked_until > 현재시간 → 423 반환 (남은 시간 포함)
4. bcrypt.compare(password, user.password_hash)
   - 불일치 → 401 반환
5. JWT access token 생성 (payload: { userId, email }, 만료: JWT_EXPIRES_IN)
6. JWT refresh token 생성 (payload: { userId }, 만료: REFRESH_EXPIRES_IN)
7. 200 + { accessToken, refreshToken } 반환

### 핵심 규칙
- 이메일 없음과 비밀번호 불일치를 동일한 에러 메시지로 응답 (보안)
- 잠금 체크가 비밀번호 체크보다 먼저 (잠금 중 비밀번호 시도 자체를 차단)

---

## 5. 에러 & 예외 처리

| 상황 | 처리 방법 | 응답 |
|------|----------|------|
| DB 조회 실패 | 로그 기록 + 500 반환 | `{ "error": "서버 오류" }` |
| bcrypt 라이브러리 에러 | 로그 기록 + 500 반환 | `{ "error": "서버 오류" }` |
| JWT 서명 실패 | 로그 기록 + 500 반환 | `{ "error": "서버 오류" }` |
| request body 파싱 실패 | 400 반환 | `{ "error": "잘못된 요청 형식" }` |

---

## 6. 보안 고려사항

- 이메일 존재 여부를 응답으로 구분할 수 없게 할 것 (열거 공격 방지)
- 비밀번호는 로그에 절대 남기지 않을 것
- rate limiting은 이 WU 범위 밖 (인프라 레벨에서 처리 권장)

---

## 7. 엣지 케이스

- email 대소문자: "User@Email.com" vs "user@email.com" → 소문자 정규화 후 비교
- locked_until이 정확히 현재 시간 → 잠금 해제로 처리 (> 비교, >= 아님)
- password_hash가 null인 사용자 (소셜 로그인 전용 계정) → 401 반환
- 동시 로그인 요청 → 별도 제한 없음 (stateless JWT)

---

## 8. 완료 조건 (Definition of Done)

- [ ] 정상 로그인 시 200 + accessToken, refreshToken 반환
- [ ] 존재하지 않는 이메일 시 401 (이메일 없다는 사실 노출 안 함)
- [ ] 비밀번호 불일치 시 401
- [ ] 잠금 계정 시 423 + lockedUntil
- [ ] 입력값 누락 시 400
- [ ] email 대소문자 정규화 처리

---

## 9. 참고

- **선행 WU**: WU-01 (JWT 생성/검증 유틸), WU-02 (users 테이블)
- **후행 WU**: WU-06 (실패 제한 — 이 API의 실패 경로에 카운트 로직 추가)
- **관련 파일(예상)**: routes/auth.ts, controllers/auth.ts, services/auth.ts