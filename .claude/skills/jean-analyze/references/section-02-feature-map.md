# Section 02: 기능 매핑 (`02-feature-map.md`)

**생성 조건**: `has_api = true` 일 때만

## 분석 순서

1. **API 엔드포인트 추출** — 프레임워크별 패턴으로 검색한다.
   - FastAPI: Grep `@router\.(get|post|put|patch|delete)` 또는 `@app\.(get|post|put|patch|delete)`
   - Flask: Grep `@app\.route|@blueprint\.route`
   - Django: `urls.py` Read → URL 패턴
   - Spring Boot: Grep `@(Get|Post|Put|Patch|Delete)Mapping|@RequestMapping`
   - Express: Grep `router\.(get|post|put|patch|delete)` 또는 `app\.(get|post|put|patch|delete)`
   - NestJS: Grep `@Get|@Post|@Put|@Patch|@Delete`
   - Next.js: `app/**/route.ts` 또는 `pages/api/` Glob
   - Gin/Echo: Grep `(GET|POST|PUT|PATCH|DELETE)\(`
   - Axum/Actix: Grep `\.route\(|\.get\(|\.post\(|#\[get|#\[post`
2. **핸들러 → 서비스 추적** — 각 엔드포인트 핸들러 Read → 호출하는 서비스 / 유즈케이스를 추적한다.
3. **서비스 → 데이터 추적** — 서비스 메서드 Read → Repository / ORM 호출을 추적한다.
4. **요청 / 응답 스키마** — 관련 DTO / Schema 파일을 매핑한다.
5. **외부 API 클라이언트** — `clients/` 또는 `adapters/` 를 스캔한다.

## 출력 포맷

<output_format>
```markdown
# 기능 매핑

> 생성일: {YYYY-MM-DD HH:MM}

## TL;DR

- 총 API 엔드포인트: {N}개
- 도메인: {기능별 그룹 목록}

## 1. API 엔드포인트 전체 목록

| # | Method | Path | 핸들러 파일:함수 | 설명 |
|---|--------|------|-----------------|------|
| {실제 코드에서 추출} | ... | ... | ... | ... |

## 2. 기능별 파일 추적 맵

### 2.N {기능명}

**진입점**: `{METHOD} {path}`

```
{핸들러}() [{파일경로}]
  ▼
{서비스/유즈케이스}() [{파일경로}]
  ▼
{데이터 접근}() [{파일경로}]
  ▼
응답
```

**관련 파일 목록**:

| 역할 | 파일 |
|------|------|
| ... | ... |

## 3. 외부 API 연동 맵 (해당 시)

| 클라이언트 | 파일 | 대상 | 사용처 |
|-----------|------|------|--------|
| ... | ... | ... | ... |
```
</output_format>
