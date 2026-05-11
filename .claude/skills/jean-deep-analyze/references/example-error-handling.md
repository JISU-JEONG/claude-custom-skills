# Worked Example: "에러 핸들링 누락" 분석

`/jean:deep-analyze` 가 어떤 결과물을 만들어야 하는지 보여주는 worked example.

가상 fixture (FastAPI bookmarks API — 8개 .py 소스, 1 test) 에 대해 "에러 핸들링 누락 어디서 발생해?" 주제로 분석한 결과 1세트.

> 이 결과는 *교육 목적의 예시*. 실제 분석 시에는 사용자 프로젝트의 진짜 코드를 Glob/Grep/Read 로 수집해 같은 형태로 채운다.

---

# 에러 핸들링 누락 심층 분석

> 생성일: 2026-04-26 | 프로젝트: fastapi-bookmarks-mini
> 스택: Python 3.12 + FastAPI 0.110 + SQLAlchemy 2.0 (async)
> 분석 질문: "에러 핸들링 누락 어디서 발생해?"
> 스캔 범위: `app/`, `tests/`

## TL;DR

- 발견 항목: 3개 (Critical 0, Warning 2, Info 1)
- 핵심 발견:
  - DB 트랜잭션 commit 시점에 try/except 없음 (Service 레이어 2건) — 글로벌 errorhandler 커버하지만 graceful 하지 않음
  - 글로벌 exception handler 자체 부재 — `app/main.py` 에 `@app.exception_handler` 없음 (Info)

## 발견 항목 요약

| # | 심각도 | 파일:라인 | 설명 | 수정 난이도 |
|---|--------|----------|------|------------|
| 1 | Warning | `app/services/bookmark_service.py:28-30` | `BookmarkService.delete` — commit 실패 시 미캐치 | Easy |
| 2 | Warning | `app/services/bookmark_service.py:18-22` | `BookmarkService.create` — IntegrityError 미처리 | Easy |
| 3 | Info | `app/main.py` (전체) | 글로벌 exception handler 부재 | Medium |

## 상세 분석

### #1. [Warning] BookmarkService.delete — commit 실패 시 미캐치

**위치**: `app/services/bookmark_service.py:28-30`

**문제 코드**:
```python
async def delete(self, bookmark_id: int) -> None:
    bm = await self.session.get(Bookmark, bookmark_id)
    if bm:
        await self.session.delete(bm)
        await self.session.commit()  # ← 실패 시 미캐치
```

**왜 문제인가**: `session.commit()` 은 DB connection 끊김·deadlock·constraint violation 등으로 실패할 수 있다. FastAPI 의 기본 글로벌 errorhandler 는 이를 500 으로 커버하나, 사용자에게는 "Internal Server Error" 만 노출되고 클라이언트가 retry 정책을 결정할 단서가 없다 (graceful 부족).

**영향 범위**: DELETE 엔드포인트 호출 시 일시적 DB 장애에 사용자가 명확한 응답을 받지 못함.

**수정 제안**:
```python
from sqlalchemy.exc import OperationalError, IntegrityError
from fastapi import HTTPException

async def delete(self, bookmark_id: int) -> None:
    bm = await self.session.get(Bookmark, bookmark_id)
    if bm:
        try:
            await self.session.delete(bm)
            await self.session.commit()
        except IntegrityError:
            await self.session.rollback()
            raise HTTPException(409, "이 북마크는 다른 항목과 연결되어 있어 삭제할 수 없습니다")
        except OperationalError:
            await self.session.rollback()
            raise HTTPException(503, "DB 일시 장애 — 잠시 후 다시 시도해 주세요")
```

**수정 난이도**: Easy

---

### #2. [Warning] BookmarkService.create — IntegrityError 미처리

**위치**: `app/services/bookmark_service.py:18-22`

**문제 코드**:
```python
async def create(self, data) -> Bookmark:
    bm = Bookmark(url=str(data.url), title=data.title, tags=data.tags)
    self.session.add(bm)
    await self.session.commit()
    await self.session.refresh(bm)
    return bm
```

**왜 문제인가**: `url` 컬럼에 unique constraint 가 추가될 가능성·DB connection 장애 시 `commit` 이 throw. #1 과 동일 패턴.

**수정 제안**: #1 과 같은 try/except 패턴 적용. (기존 헬퍼 부재 — 새로 작성. 향후 `_safe_commit(session)` 헬퍼 추출 후보)

**수정 난이도**: Easy

---

### #3. [Info] 글로벌 exception handler 부재

**위치**: `app/main.py` (전체)

**문제 코드**: 해당 항목 부재 (`@app.exception_handler` 데코레이터 없음).

**왜 문제인가**: FastAPI 의 기본 핸들러가 모든 미캐치 예외를 500 으로 처리하나, 로깅·알림·사용자 친화 메시지가 통일되지 않는다.

**수정 제안**:
```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.exception("unhandled error: %s %s", request.url, exc)
    return JSONResponse(status_code=500, content={"detail": "내부 오류"})
```

**수정 난이도**: Medium (로깅 설정·구조화된 에러 응답 표준 결정 필요)

## 검증 과정에서 확인된 양호 패턴

- `app/api/bookmarks.py:36-39` — `get_bookmark` 핸들러는 None 반환 시 `HTTPException(404)` 명시. ✓
- Pydantic 입력 검증 — `BookmarkCreate` 가 자동으로 422 처리. ✓
- 글로벌 errorhandler 부재이긴 하나 FastAPI 기본 처리가 500 보장 — *완전 미커버* 가 아니라 *graceful 부족*. → 발견 항목 #1·#2 의 심각도가 Critical 이 아니라 Warning 인 근거.

---

> 검증: 2회 수행 | 제거 1건 (`api/bookmarks.py:get_bookmark` 가 N+1 false positive 였음 — 단건 조회), 수정 0건, 추가 1건 (#3 글로벌 핸들러)
