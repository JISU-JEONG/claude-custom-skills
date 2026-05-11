# Section 05: 프로젝트 건강도 (`05-health.md`)

**생성 조건**: 항상 (모든 프로젝트에 해당)

## 분석 순서

1. **파일 통계** — 언어별 확장자로 소스 / 테스트 파일 수를 집계한다.
   - Python: `*.py`
   - TypeScript / JavaScript: `*.ts`, `*.tsx`, `*.js`, `*.jsx`
   - Go: `*.go`
   - Rust: `*.rs`
   - Java / Kotlin: `*.java`, `*.kt`
2. **LOC 집계**
   ```bash
   find {소스루트} -name '*.{ext}' \
     -not -path '*node_modules*' -not -path '*__pycache__*' -not -path '*.git*' \
     -exec cat {} + | wc -l
   ```
3. **TODO / FIXME 현황** — Grep(`TODO|FIXME|HACK|XXX`, path={소스루트}) → output_mode=content
4. **테스트 매핑** — 소스 파일 vs 테스트 파일 대응 현황을 표로 정리한다.
5. **주요 의존성** — 패키지 매니저별 목록을 추출한다.
   - Python: `pyproject.toml` Read 또는 `pip list 2>/dev/null | head -30`
   - Java / Kotlin: `build.gradle` 또는 `pom.xml` Read → dependencies
   - Node.js: `package.json` Read → dependencies + devDependencies
   - Go: `go.mod` Read → require
   - Rust: `Cargo.toml` Read → dependencies
6. **업데이트 가능한 패키지 (선택)** — 각 패키지 매니저의 outdated 명령. 실패 시 스킵.
   - `poetry show --outdated`
   - `npm outdated`
   - `go list -m -u all`
   - `./gradlew dependencyUpdates`

## 출력 포맷

<output_format>
```markdown
# 프로젝트 건강도

> 생성일: {YYYY-MM-DD HH:MM}

## TL;DR

- 소스 파일: {N}개 | 테스트 파일: {N}개
- 총 코드 라인: ~{N} LOC
- TODO/FIXME: {N}개
- 테스트 매핑률: {N}%

## 1. 파일/모듈 통계

| 디렉토리 | 파일 수 | 역할 |
|----------|---------|------|
| {프로젝트 구조에 맞게 동적 생성} | ... | ... |

## 2. TODO/FIXME 현황

| # | 유형 | 파일:라인 | 내용 |
|---|------|----------|------|
| {Grep 결과} | ... | ... | ... |

## 3. 테스트 매핑 현황

| src 모듈 | 대응 테스트 | 상태 |
|----------|-----------|------|
| ... | ... | ... |

## 4. 주요 의존성

| 패키지 | 버전 | 용도 |
|--------|------|------|
| ... | ... | ... |
```
</output_format>
