# Section 04: 데이터 모델 분석 (`04-data-model.md`)

**생성 조건**: `has_data_layer = true` 일 때만

## 분석 순서

1. **모델 / 엔티티 파일 탐색** — ORM 별 패턴.
   - SQLAlchemy: `models/**/*.py` Grep `Column|relationship|__tablename__`
   - Django: `models.py` Grep `models.Model`
   - JPA / Hibernate: Grep `@Entity|@Table|@Column|@ManyToOne|@OneToMany`
   - Exposed (Kotlin): Grep `object.*Table|\.integer\(|\.varchar\(`
   - Prisma: `prisma/schema.prisma` Read
   - Drizzle: `schema.ts` Read
   - TypeORM: Grep `@Entity|@Column`
   - GORM: Grep `gorm.Model`
   - Diesel (Rust): `schema.rs` Read, Grep `table!`
   - 기타: `models/`, `entities/`, `domain/` Glob
2. **각 모델 Read** — 필드 / 타입 / 관계(FK) / 제약조건을 추출한다.
3. **JSON / JSONB / 비정형 필드 식별** — 구조를 추정해 별도 표시한다.
4. **마이그레이션 파일 존재 확인** — `alembic/`, `migrations/`, `prisma/migrations/`, `flyway/`, `liquibase/`, `db/migrate/`.

## 출력 포맷

<output_format>
```markdown
# 데이터 모델 분석

> 생성일: {YYYY-MM-DD HH:MM}

## TL;DR

- ORM: {ORM명}
- 핵심 엔티티: {목록}
- 비정형 데이터 필드: {있으면 목록}

## 1. 엔티티 관계도

```
{텍스트 ER 다이어그램}
```

## 2. 엔티티 상세

### 2.N {엔티티명}

| 필드 | 타입 | 제약조건 | 설명 |
|------|------|---------|------|
| {실제 코드에서 추출} | ... | ... | ... |
```
</output_format>
