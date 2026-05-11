---
name: cal
description: 일정·약속·미팅 관리. 추가·조회·삭제·고정 명령어 지원. 자연어 날짜 (내일/모레/오늘) 처리. 데이터는 `~/.jean/schedule/schedules.json` 에 저장 — 외부 status line 등이 참조 가능. 사용자가 일정을 잡거나 약속을 추가하거나 다가오는 미팅을 확인하려 할 때 호출. 키워드 — 일정, 약속, 미팅, 캘린더, schedule, calendar, appointment, remind, meeting.
allowed-tools: Read, Write, Edit, Bash
argument-hint: "add 내일 14:00 팀미팅 | list [--all] | rm <id> | pin <id> | unpin <id> | past"
---

# Schedule Manager

일정을 `~/.jean/schedule/schedules.json`에서 관리한다.

## 데이터 파일

- 활성 일정: `~/.jean/schedule/schedules.json`
- 지난 일정 archive: `~/.jean/schedule/archive.json` (자동 정리 시 이전됨)

```json
{
  "schedules": [
    { "id": "a1b2c3", "date": "2026-04-17", "time": "14:00", "title": "팀미팅", "pinned": true }
  ]
}
```

archive.json도 동일 스키마 — `schedules` 배열에 객체 누적.

## 명령어 처리

`$ARGUMENTS`를 파싱하여 아래 규칙대로 처리한다.

### 1. `add <date> <time> <title>`

예시: `/cal add 2026-04-17 14:00 팀미팅`

- date 형식: `YYYY-MM-DD`
- time 형식: `HH:MM` (24시간)
- title: 나머지 전체 문자열
- id: 6자리 hex (`openssl rand -hex 3` 으로 생성)
- pinned: 기본값 false (생략 가능)
- schedules.json을 읽고, schedules 배열에 추가한 뒤 저장
- 출력 템플릿: `등록됨 — [a1b2c3] 2026-04-17 14:00 팀미팅`

**편의 기능:**
- 날짜 없이 시간+제목만 입력하면 (`/cal add 14:00 팀미팅`) 오늘 날짜로 자동 설정
- "내일", "tomorrow" → 내일 날짜로 변환
- "모레" → 모레 날짜로 변환

### 2. `list`

예시: `/cal list`

- schedules.json을 읽고 날짜+시간 순으로 정렬
- 고정된 일정은 📌 표시
- 다가오는 일정만 보여주되, `--all` 플래그가 있으면 전체 표시

출력 형식:
```
다가오는 일정
─────────────
1. 📌 2026-04-17 14:00 팀미팅 (a1b2c3)
2. 2026-04-18 10:00 코드리뷰 (d4e5f6)
```

id는 제목 뒤 괄호 안에 표시한다.

### 3. `rm <id>`

예시: `/cal rm a1b2c3` 또는 `/cal rm a1b` (prefix)

- id 매칭은 prefix 허용. `a1b` 입력 시 `a1b` 로 시작하는 모든 항목 검색
- 후보 1개 → 삭제. 0개 → 에러 메시지. 2개 이상 → 후보 모두 표시 후 사용자에게 정확한 id 요청 (자동 선택 안 함)
- 삭제 전 해당 일정 정보 표시
- 출력 템플릿: `삭제됨 — [a1b2c3] 2026-04-17 14:00 팀미팅`

### 4. `pin <id>`

예시: `/cal pin a1b2c3`

- id 매칭은 `rm` 과 동일 (prefix 허용, 모호 시 후보 표시)
- 해당 일정에 `"pinned": true` 설정
- pin 의 효과는 list 의 "자동 정리 정책" 참고 (기간 지나도 보존)
- 출력 템플릿: `📌 고정됨 — [a1b2c3] 2026-04-17 14:00 팀미팅`

### 5. `unpin <id>`

예시: `/cal unpin a1b2c3`

- id 매칭은 `rm` 과 동일
- 해당 일정에서 `pinned` 필드 제거 (또는 `false` 설정)
- 출력 템플릿: `고정 해제됨 — [a1b2c3] 2026-04-17 14:00 팀미팅`

### 6. `past`

예시: `/cal past`

지난 일정을 모아 출력한다. 두 가지 소스를 합친다:
1. `archive.json` 의 모든 항목 (날짜가 지나 자동 이전된 일정)
2. `schedules.json` 의 오늘 일정 중 `time < 현재시각` 인 것 (시간이 이미 지난 오늘 일정)

- 정렬: 가장 최근 → 가장 오래된 순 (date desc, time desc)
- archive.json이 없으면 1번은 빈 목록으로 처리 (에러 X)
- 둘 다 비어있으면 `지난 일정 없음` 출력
- 출력 형식:
```
지난 일정
─────────────
1. 2026-05-02 14:00 거실 청소 (d33a3d)
2. 2026-05-01 18:00 회의 (a1b2c3)
```

### 7. 인수 없이 호출

`/cal` 만 입력하면 `list`와 동일하게 동작한다.

## 자동 정리 정책

자동 정리는 `list` 호출 시점과 외부 status line 스크립트 호출 시점에 일어난다. add/rm/pin/unpin 시점에는 데이터를 건드리지 않는다.

list 호출 시:
- `pinned: true` 가 아니고 `date < 오늘` 인 항목은 `schedules.json` 에서 제거하고 `archive.json` 에 append (이전)
- `--all` 플래그가 붙으면 정리도 안 하고 전체 표시 (감사·복구용)
- archive.json은 `past` 명령어로 조회

"다가오는 일정" 의 정의 — `date >= 오늘`. 오늘 지난 시간이라도 오늘 일정은 표시한다.

## status line 연동

이 스킬은 데이터 CRUD 만 담당. status line 표시 자체는 별도 설정 (`~/.claude/settings.json` 의 `statusLine` 또는 외부 스크립트) 에서 `~/.jean/schedule/schedules.json` 을 읽어 처리한다. 이 스킬의 책임은 그 데이터 파일을 일관된 형태로 유지하는 것까지.

## 주의사항

- schedules.json이 없으면 `{ "schedules": [] }`로 새로 생성
- `~/.jean/schedule/` 디렉토리가 없으면 `mkdir -p`로 생성
- JSON 파일 쓸 때 2-space indent로 포맷팅
- 오늘 날짜는 !`date +%Y-%m-%d` 로 확인
