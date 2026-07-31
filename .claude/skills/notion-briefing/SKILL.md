---
name: notion-briefing
description: Notion Tasks DB 기반 브리핑 스킬 (클라우드 루틴용). 프롬프트로 지정된 브리핑 타입(morning/midday/evening/midnight)과 Discord 웹훅 URL을 받아, 아침 전체 브리핑 또는 My Day 리마인드를 조회해 Discord로 전송한다.
---

# Notion 브리핑 스킬 (클라우드 루틴 버전)

이 스킬은 `/schedule` 클라우드 루틴에서 실행되는 것을 전제로 한다 (로컬 `ntn` CLI, `sessions_send`
없음). 대신 **연결된 Notion MCP 커넥터**로 조회하고, **Discord 웹훅**으로 전송한다.

## 이 스킬을 호출할 때 프롬프트에 반드시 포함되어야 하는 것

1. **브리핑 타입**: `morning` | `midday` | `evening` | `midnight`
2. **Discord 웹훅 URL**: 이 값은 절대 이 스킬 파일이나 저장소에 하드코딩하지 않는다.
   매 실행마다 루틴 프롬프트에서 전달받아서만 사용한다 (git에 웹훅 URL이 남지 않도록).

프롬프트에 이 두 가지가 없으면 작업을 진행하지 말고 무엇이 빠졌는지 보고하고 종료한다.

## 브리핑 타입별 동작

| 타입 | 보통 트리거 시각 (KST) | 동작 |
|---|---|---|
| `morning` | 07:00 | To Do 전체 due 카테고리별 브리핑 |
| `midday` | 12:00 | My Day 미완료 리마인드 |
| `evening` | 19:00 | My Day 미완료 리마인드 |
| `midnight` | 23:30 | My Day 미완료 리마인드 |

## 오늘 날짜 (KST) 구하기

이 컨테이너는 UTC 기준이다. 사용자 로컬(Asia/Seoul, UTC+9) 날짜를 써야 하므로 Bash로:

```bash
date -u -d '+9 hours' +'%Y-%m-%d %A'
```

로 KST 기준 오늘 날짜와 요일을 구한다. 이후 모든 "오늘" 비교는 이 KST 날짜를 기준으로 한다.

## Notion 설정

- Tasks DB data source URL: `collection://2ce1232d-c5c0-81be-a31d-000b9c324199`
- 조회 도구: Notion MCP 커넥터의 `notion-query-data-sources` (SQL 모드)
- 속성: Name(title), Status(To Do/Doing/Done), My Day(checkbox), Due(date)
- 체크박스 비교는 파라미터 값 `"__YES__"` / `"__NO__"` 사용

## 공통: 태스크 조회

### To Do 전체 조회 (morning용)

```json
{
  "data": {
    "mode": "sql",
    "data_source_urls": ["collection://2ce1232d-c5c0-81be-a31d-000b9c324199"],
    "query": "SELECT * FROM \"collection://2ce1232d-c5c0-81be-a31d-000b9c324199\" WHERE Status = ? LIMIT 100",
    "params": ["To Do"]
  }
}
```

### My Day 미완료 조회 (midday/evening/midnight용)

```json
{
  "data": {
    "mode": "sql",
    "data_source_urls": ["collection://2ce1232d-c5c0-81be-a31d-000b9c324199"],
    "query": "SELECT * FROM \"collection://2ce1232d-c5c0-81be-a31d-000b9c324199\" WHERE \"My Day\" = ? AND Status != ? LIMIT 100",
    "params": ["__YES__", "Done"]
  }
}
```

---

## morning — 아침 브리핑

### 분류 기준 (오늘 날짜 = 위에서 구한 KST 오늘)

| 카테고리 | 조건 | 이모지 |
|---|---|---|
| 오늘 마감 | due == today | ☀️ |
| 기한 초과 | due < today | 🔴 |
| 이번 주 마감 | today < due ≤ today+7 | 📅 |
| 향후 예정 | due > today+7 | 🗓️ |
| Due 없음 | due 없음 | 📌 |

### 출력 형식

```
🔄 Routina

📋 아침 브리핑 — YYYY년 M월 D일 (요일)

---

☀️ 오늘 마감 (N개)
• 태스크명

🔴 기한 초과
없음 ✅

📅 이번 주 마감 (N개)
• 태스크명 — M월 D일 (요일)

🗓️ 향후 예정
• 태스크명 — M월 D일

📌 Due 없음 (N개)
• 태스크명

---

오늘 하루도 잘 해내세요! 💪
```

**규칙:**
- 항목 없는 카테고리: 헤더 + "없음 ✅"
- 이번 주 마감은 날짜(요일) 함께 표시
- To Do가 0개면: `오늘은 To Do 태스크가 없습니다 🎉` 만 담아서 전송 (아래 전송 규칙 참고)

---

## midday / evening — My Day 리마인드

### 조건

- My Day 미완료 항목이 **1개 이상일 때만** 전송
- 0개면 Discord 웹훅 호출 자체를 하지 않고 조용히 종료

### 출력 형식

```
🔄 Routina

⏰ [점심 전 | 저녁 전] 리마인드 — M월 D일 (요일)

My Day 미완료 태스크 N개 남아있어요:
• 태스크명
• 태스크명

파이팅! 🙌
```

---

## midnight — 자정 리마인드

### 조건

- My Day 미완료 항목이 **1개 이상일 때만** 전송
- 0개면 Discord 웹훅 호출 자체를 하지 않고 조용히 종료

### 출력 형식

```
🔄 Routina

🌙 자정 리마인드 — M월 D일 (요일)

My Day 미완료 태스크 N개가 남아있어요:
• 태스크명
• 태스크명
```

---

## 전송 방법 (Discord 웹훅)

1. 위 형식대로 메시지 텍스트를 완성한다.
2. Write 도구로 임시 파일(예: `discord_payload.json`)에 아래 형태의 유효한 JSON을 작성한다
   (줄바꿈은 `\n`으로, 따옴표는 `\"`로 정확히 이스케이프):
   ```json
   {"content": "완성된 메시지 텍스트"}
   ```
3. Bash로 전송한다 (WEBHOOK_URL은 프롬프트에서 받은 값 그대로 사용, 절대 로그/커밋하지 않음):
   ```bash
   curl -s -X POST -H "Content-Type: application/json" \
     --data @discord_payload.json \
     "<프롬프트에서 받은 웹훅 URL>"
   ```
4. 전송 후 임시 파일은 삭제한다.

## 에러 처리

- Notion 조회 실패 시: 그 사실을 요약해서 Discord 웹훅으로 `❌ Notion 조회 실패: {에러 요약}` 전송
- 웹훅 URL이 프롬프트에 없으면: 아무것도 전송하지 말고 무엇이 빠졌는지만 보고하고 종료
