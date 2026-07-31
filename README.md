# routina

`/schedule`(Claude Code 클라우드 루틴) 기능을 테스트하기 위한 프로젝트.

## 목적

클라우드 루틴(RemoteTrigger로 생성되는 CCR 세션)이 저장소에 포함된
**커스텀 스킬**을 실제로 로드해서 실행할 수 있는지 확인하는 것이 1차 목표.

## 테스트 계획

1. `.claude/skills/hello-test/SKILL.md` — 최소 동작 확인용 더미 스킬
   (호출되면 고정 문자열 + 타임스탬프만 출력)
2. 이 저장소를 GitHub에 올린 뒤, 그 URL을 `job_config.ccr.session_context.sources`로
   지정한 1회성(run_once_at) 루틴을 생성
3. `allowed_tools`에 `Skill` 포함
4. 루틴 프롬프트: "hello-test 스킬을 사용해서 결과를 보고해줘"
5. 실행 후 `https://claude.ai/code/routines/{ROUTINE_ID}`에서 결과 확인

## 확장 아이디어 (테스트 통과 시)

- 매일 영어 단어장 파일 생성/갱신 (Google Drive 커넥터 또는 이 저장소에 커밋)
- 결과를 Discord 웹훅으로 전송
