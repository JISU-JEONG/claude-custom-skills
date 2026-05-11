# {wu_id}: {wu_title}

> 난이도: {difficulty} | 의존관계: {dependencies}

---

## 1. 개요

{description}

---

## 2. API 스펙

> 이 섹션은 API 관련 WU에만 작성. 설정/DB/인프라 WU는 "해당 없음"으로 표기하고 생략.

### {method} {path}

**Request:**
- Headers: {headers}
- Body:
```json
{request_body_schema}
```
- Validation Rules:
{validation_rules}

**Response:**

| Status | 조건 | Body |
|--------|------|------|
| {success_code} | 정상 처리 | `{success_body}` |
| {error_code_1} | {error_condition_1} | `{error_body_1}` |
| {error_code_2} | {error_condition_2} | `{error_body_2}` |

---

## 3. 데이터 모델

> 이 섹션은 DB 관련 WU에만 작성. API-only WU는 조회할 테이블/필드만 참조로 기술.

### 테이블/컬렉션: {table_name}

| 컬럼 | 타입 | 제약조건 | 설명 |
|------|------|---------|------|
| {column} | {type} | {constraints} | {description} |

### 마이그레이션
```sql
{migration_sql}
```

---

## 4. 비즈니스 로직

### 처리 흐름
1. {step_1}
2. {step_2}
3. {step_3}

### 핵심 규칙
- {rule_1}
- {rule_2}

---

## 5. 에러 & 예외 처리

| 상황 | 처리 방법 | 응답 |
|------|----------|------|
| {situation_1} | {handling_1} | {response_1} |
| {situation_2} | {handling_2} | {response_2} |

---

## 6. 보안 고려사항

- {security_item_1}
- {security_item_2}

---

## 7. 엣지 케이스

- {edge_case_1}
- {edge_case_2}

---

## 8. 완료 조건 (Definition of Done)

- [ ] {dod_1}
- [ ] {dod_2}
- [ ] {dod_3}

---

## 9. 참고

- **선행 WU**: {prerequisite_wus}
- **후행 WU**: {dependent_wus}
- **관련 파일(예상)**: {expected_files}