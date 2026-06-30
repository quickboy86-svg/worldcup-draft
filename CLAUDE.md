# worldcup-draft

2026 월드컵 드래프트 + 결과 추적 사이트. 4명이 각각 3개 조를 드래프트로 선택하고, 선택한 팀의 토너먼트 진출 성과에 따라 점수를 받는다.

## 파일 구조

```
draft.html        — 드래프트 진행 화면 (조 선택 UI)
result.html       — 메인 결과 페이지 (조별 순위 + 토너먼트)
index.html        — draft.html로 리다이렉트 (또는 진입 페이지)
draft-result.json — 드래프트 결과 저장 (result.html이 읽어들임)
api-data.json     — GitHub Actions가 매시간 업데이트 (수동 편집 금지)
.github/workflows/update-football-data.yml — 자동 업데이트 워크플로
```

## 핵심 데이터 구조

**`draft-result.json`** — PARTICIPANTS의 소스
```json
[
  { "name": "영신", "colorIdx": 0, "groups": ["A", "B", "C"] },
  ...
]
```
result.html이 로드 시 fetch해서 `PARTICIPANTS` 배열에 채운다.

**`GROUPS`** (result.html 상단에 하드코딩)
```js
{ id: 'A', teams: ['팀1', '팀2', '팀3', '팀4'], matches: [...] }
```

**`KNOCKOUT_SCHEDULE`** — 32강 대진 레이블 배열 (48개 조별 경기 후 순서대로)
```js
['A조 1위', 'B조 2위', '3위(A·B·C·D)', ...]
```

**`COLOR_HEX`** — 참가자 색상 (colorIdx와 1:1 매핑)
```js
['#FF5252','#00BCD4','#FFD54F','#AB47BC']
```

## 점수 체계

| 이벤트 | 점수 |
|--------|------|
| 조 1위·2위 진출 | +1점 |
| 3위 중 상위 8팀 진출 | +1점 |
| 32강 통과 (16강 진출) | +2점 |
| 16강 통과 (8강 진출) | +3점 |
| 8강 통과 (4강 진출) | +4점 |

`teamRoundPoints` Map에 팀명 → 누적점수 저장. `getParticipantScore(p)`가 해당 참가자가 선택한 조의 모든 팀 점수 합산.

## API 데이터 quirks

**football-data.org v4 API:**
- `GET /competitions/WC/matches` — 전체 경기. 단, 토너먼트 경기는 `homeTeam`/`awayTeam`이 null로 오는 경우 있음
- `GET /competitions/WC/matches?stage=LAST_32` — 32강만 별도 fetch. 팀 정보가 더 완전함
- GitHub Actions에서 LAST_32를 별도로 fetch해서 전체 데이터에 merge한 뒤 api-data.json에 저장

**PK 경기 스코어 버그:**
- `score.fullTime`에는 PK 골이 누적됨 (예: 1-1 AET + PK 4-3이면 fullTime이 5-4로 옴)
- 올바른 표시: `score.regularTime.home + score.extraTime.home` vs away
- `score.duration === 'PENALTY_SHOOTOUT'`일 때 이 방식 사용
- PK 결과 별도 표시: `score.penalties.home : score.penalties.away`

## 토너먼트 브래킷 로직

`renderTournament(allMatches)` 함수:

1. **32강 팀 결정**: API의 homeTeam/awayTeam이 있으면 사용, 없으면 `resolveLabel(KNOCKOUT_SCHEDULE[idx])`로 순위표에서 계산
2. **16강 이후**: `computeFromPrev(stage, prevStage)` — 이전 단계 경기를 날짜순 정렬 후 연속쌍(0+1, 2+3...)의 승자로 채움
3. **3·4위전**: SF 두 경기의 패자
4. **결승**: SF 두 경기의 승자

`resolveLabel(label)`:
- `"A조 1위"` → `groupTeamMap['A_1']` 조회
- `"3위(A·B·C·D)"` → `top8Third` 배열에서 해당 그룹 팀 검색

`buildGroupTeamMap(standingsData)`: 순위표 API 데이터로 `groupTeamMap`, `top8Third` 생성

## 로컬 개발 서버

```powershell
# 포트 8080에서 실행
Start-Process python -ArgumentList '-m','http.server','8080' -WorkingDirectory 'c:\Projects\worldcup-draft'
# 브라우저: http://localhost:8080/result.html
```

## Git 작업 주의사항

**GitHub Actions conflict**: Actions가 매시간 api-data.json을 커밋 → 로컬 push가 reject될 수 있음

```bash
# push 전 항상:
git fetch origin && git rebase origin/main && git push

# conflict 발생 시:
git checkout --theirs api-data.json
git add api-data.json
git rebase --continue
git push
```

## 참가자 추가/변경 방법

`draft-result.json` 수정:
```json
[
  { "name": "이름", "colorIdx": 0, "groups": ["A","B","C"] }
]
```
- `colorIdx`: 0~3 (COLOR_HEX 배열 인덱스)
- `groups`: 선택한 조 ID 배열 (각 참가자 3개)
- 참가자 수가 4명이면 `colorIdx` 0-3, `COLOR_HEX`도 4색

## 조별리그 데이터 수정 방법

`result.html` 상단 `GROUPS` 배열에서 직접 수정.
팀명은 한국어로 하드코딩됨 — `toKo(teamObj)` 함수가 API 영문명 → 한국어 변환.

`toKo()` 번역 테이블이 없는 팀이 추가되면 해당 함수에 매핑 추가 필요.

## 탭 구조

```
탭0: 조별리그 (기본)
탭1: 토너먼트 (KO 경기가 1개 이상 존재하면 자동 전환)
```

`autoSwitchedToTournament` 플래그로 최초 1회만 자동 전환.

## 배포

GitHub Pages (`main` 브랜치 루트). push하면 1-3분 후 반영.
브라우저 캐시 때문에 변경사항 안 보이면 Ctrl+Shift+R.
