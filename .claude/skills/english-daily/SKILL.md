---
name: english-daily
description: 매일 영어 문장 5개 + 단어 10개를 학습하고 1일/3일/7일 전 학습한 문장·단어를 복습하는 스킬 (클라우드 루틴용). Discord 웹훅 URL을 프롬프트로 받아 결과를 전송한다.
---

# 영어 데일리 학습 스킬 (클라우드 루틴 버전)

이 스킬은 `/schedule` 클라우드 루틴에서 매일 실행되는 것을 전제로 한다. 진행 상황은 Notion
데이터베이스에 기록하고, 결과는 Discord 웹훅으로 전송한다.

## 이 스킬을 호출할 때 프롬프트에 반드시 포함되어야 하는 것

**Discord 웹훅 URL** — 이 값은 절대 이 스킬 파일이나 저장소에 하드코딩하지 않는다. 매 실행마다
루틴 프롬프트에서 전달받아서만 사용한다 (git에 웹훅 URL이 남지 않도록).

프롬프트에 웹훅 URL이 없으면 작업을 진행하지 말고 무엇이 빠졌는지 보고하고 종료한다.

## 데이터 파일

이 스킬과 같은 저장소에 커밋되어 있는 정적 데이터를 읽는다 (외부 API 호출 없음). 출처는 Oxford
Phrase List(750개 표현, A1~C1)와 The Oxford 3000 + The Oxford 5000 by CEFR level(A1~C1,
5320단어)이며, 둘 다 영어 원문만 있고 한글 뜻은 들어있지 않다 (아래 "번역" 항목 참고):

- 문장/표현: `.claude/skills/english-daily/data/sentences.json` (750개)
- 단어: `.claude/skills/english-daily/data/words.json` (5320개)

형식 (둘 다 `id` 오름차순 = 학습 순서. 템플릿은 같은 폴더의 `sentences.example.json` /
`words.example.json` 참고):

```json
// sentences.json
[{ "id": 1, "en": "a few", "level": "A1", "examples": ["a few minutes", "a few times"] }]

// words.json
[{ "id": 1, "en": "apple", "level": "B2" }]
```

`level`(CEFR 레벨)은 참고용 메타데이터일 뿐 스킬 동작에는 쓰이지 않음 — 없어도 지장 없음.
`examples`(문장에만 존재, 선택 필드)는 원본 PDF에서 그 표현 아래에 들여쓰기로 딸려 있던 사용
예문/관련 표현이다. 학습 메시지에 표현을 보여줄 때 있으면 참고용으로 1~2개 같이 보여줘도 좋다.

파일이 없거나 비어 있으면: Discord로 `❌ 데이터 파일 없음: {경로}` 전송 후 종료.

## 번역 (한글 뜻)

데이터 파일에 한글 뜻이 없으므로, **네가 직접 실행 시점에 자연스러운 한국어로 번역**해서 쓴다
(별도 번역 API 호출 없음). 신규로 배우는 문장/단어를 Progress DB에 기록할 때 그 번역을
`Meaning`에 같이 저장해두면, 나중에 복습할 때는 다시 번역할 필요 없이 그때 저장해둔 `Meaning`을
그대로 읽어서 쓰면 된다 (매번 재번역하면 같은 문장인데 표현이 달라질 수 있으므로).

## 오늘 날짜 (KST) 구하기

컨테이너는 UTC 기준이므로 Bash로:

```bash
date -u -d '+9 hours' +'%Y-%m-%d %A'
```

로 KST 기준 오늘 날짜와 요일을 구한다. 이후 모든 "오늘" 비교는 이 KST 날짜를 기준으로 한다.

## Notion 설정

- Progress DB data source URL: `collection://170eb259-2eb6-4dbe-a1d1-43e8ffe00399`
- 조회/기록 도구: Notion MCP 커넥터의 `notion-query-data-sources`(SQL 모드), `notion-create-pages`
- 컬럼: `Type`(select: `Sentence` / `Word`), `Item ID`(number), `Text`(title, 영어 원문),
  `Meaning`(text, 한글 뜻), `Learned Date`(date)

> Progress DB가 아직 없으면 이 스킬을 실행하기 전에 먼저 생성해야 한다 (일회성 설정이며, 루틴이
> 매번 실행할 때마다 하는 작업이 아니다).

## 처리 순서

1. 오늘 날짜(KST) 계산.
2. Progress DB에서 `Type = 'Sentence'`인 행 중 `Item ID` 최댓값 조회 → 없으면 0. 다음 학습 시작
   ID = 최댓값 + 1.
3. 동일하게 `Type = 'Word'`의 `Item ID` 최댓값 조회 → 다음 학습 시작 ID.
4. `sentences.json`에서 시작 ID부터 5개 slice (문장 데이터가 소진되어 5개보다 적게 남았으면 있는
   만큼만, 0개면 "전체 학습 완료" 처리).
5. `words.json`에서 시작 ID부터 10개 slice (동일하게 소진 처리).
6. Progress DB에서 `Type = 'Sentence' AND Learned Date IN (오늘-1일, 오늘-3일, 오늘-7일)` 조회 →
   복습 문장 목록.
7. Progress DB에서 `Type = 'Word' AND Learned Date IN (오늘-1일, 오늘-3일, 오늘-7일)` 조회 → 복습
   단어 목록.
   (문장과 단어 모두 동일한 1일/3일/7일 주기로 복습한다.)
8. 4, 5에서 고른 신규 항목들 각각에 대해 자연스러운 한국어 번역을 생성한 뒤, Progress DB에 새
   페이지로 기록한다 (`Type`, `Item ID`, `Text`=en, `Meaning`=방금 생성한 번역, `Learned Date`=오늘).
9. 6, 7에서 조회한 복습 항목들은 재번역하지 않고 Progress DB에 저장되어 있던 `Meaning`을 그대로
   사용한다.
10. 아래 형식으로 Discord 메시지를 조립해 전송한다.

## 출력 형식

```
🔄 Routina — 영어 학습

📅 M월 D일 (요일)

---

📝 오늘의 문장 (N개)
1. Could you pass me the salt?
   → 소금 좀 건네주시겠어요?
...

🔤 오늘의 단어 (N개)
1. apple — 사과
...

---

⏪ 복습 · 1일 전
문장: • ... (없으면 "없음")
단어: • ... (없으면 "없음")

⏪ 복습 · 3일 전
(위와 동일 형식)

⏪ 복습 · 7일 전
(위와 동일 형식)

---

오늘도 화이팅! 💪
```

**규칙:**
- 문장/단어 데이터가 모두 소진되면 해당 섹션은 `🎉 모든 [문장|단어] 학습 완료!`로 대체한다.
- 복습 항목이 세 구간 모두 없어도 브리핑 자체는 항상 전송한다 (신규 학습 내용은 항상 있으므로).

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

- 데이터 파일이 없거나 비어 있으면: 위 "데이터 파일" 섹션 참고. 부분 진행 없이 종료.
- Notion 조회/기록 실패 시: 그 사실을 요약해서 Discord 웹훅으로 `❌ Notion 처리 실패: {에러 요약}`
  전송.
- 웹훅 URL이 프롬프트에 없으면: 아무것도 전송하지 말고 무엇이 빠졌는지만 보고하고 종료.
