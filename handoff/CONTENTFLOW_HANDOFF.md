# ContentFlow 인수인계

최종 갱신: 2026-10-13 (클라우드 세션)
다음 세션은 이 문서부터 읽고 시작한다. 내용이 바뀌면 이 문서를 고쳐서 같은 경로에 커밋한다.

---

## 1. 한눈에

- 강연 AI 처리(LectureScribe → Claude Sonnet 5.5)는 **운영 가능한 상태**다. GIN2026 강연 22건을 모두 처리하고 발행했다.
- 실행 스위치는 **OFF**다. 열린 run과 쓰지 않은 승인은 없다. 예약된 감시(트리거)도 없다.
- 10월 AI 비용 누적은 **$14.57**이다(live run의 `cost_usd` 합계).
- 강연은 ContentFlow의 일부다. 웹 글, 논문, 스레드, 북마크, PDF 수집과 요약, 분류, 임베딩, 지식베이스도 있다. 이 부분들의 완성도를 계속 높여 가야 한다(6장, 7장).
- Notion 현황판의 ContentFlow 행: https://app.notion.com/p/3f8bf0ca3d6981128a4bd23701fe1fd9 (맥미니가 세션을 마칠 때마다 갱신한다)

## 2. 구성과 역할 분담

| 구분 | 내용 |
|---|---|
| 소스 | 맥미니 외장 SSD `/Volumes/JBH_ExternalSSD/Codex/Contentflow` (main). GitHub `ZZangbo/Contentflow`에는 소스가 없고 보고서와 이 문서만 있다. |
| API | Cloudflare Worker `https://contentflow-api.jbh8284.workers.dev`. **Workers Paid**(2026-10-13 전환). `wrangler.toml`의 `[placement] region = "aws:us-east-1"`(fetch 핸들러 전용). |
| 일괄 실행기 | Durable Object `LectureBatchRunner`, `jurisdiction("us")`. cron(5분)은 대상 선정과 claim만 하고, 단계 실행은 DO 안에서 한다. |
| 웹 | Vercel `https://contentflow-xi.vercel.app` (프로젝트 contentflow). 브라우저가 workers.dev를 직접 호출한다. |
| DB | Supabase 프로젝트 `wlvhmuudjylaayunnkop` (서울 리전). **TCIM 프로젝트(`tcim_*` 테이블)와 같은 DB를 쓰니 tcim 테이블은 건드리지 않는다.** |
| 맥미니 Claude Code | 코드 수정, 테스트, 커밋, 배포, 스위치 전환, wrangler tail, Notion 현황판 갱신 |
| 클라우드 세션 | Supabase MCP로 DB 조회와 점검, 마이그레이션 검토, 실행 중 감시(필요하면 SQL 일시정지만), 맥미니에 넣을 요청문 작성. workers.dev와 vercel.app에는 접속할 수 없다. |

맥미니 터미널을 다시 열 때:
```
cd /Volumes/JBH_ExternalSSD/Codex/Contentflow
claude --continue
```

## 3. live 처리 운영 규칙

### 기본 규칙
- 승인은 실행 스위치 **OFF**일 때만 만들 수 있고, 실행은 **ON**일 때만 된다.
- 스위치는 `apps/api/wrangler.toml`의 `LECTURE_LIVE_EXECUTION_ENABLED`다. 맥미니가 바꿔 **Worker만** 배포하고, 이 변경은 커밋하지 않는다. 작업이 끝나면 반드시 OFF로 되돌려 배포한다.
- 실행 중에는 다른 Worker 배포를 하지 않는다.
- 단건 승인은 6시간 안에 첫 실행을 시작해야 하고, 일괄 승인은 24시간이다. 한 번 시작한 run은 만료와 무관하게 재개할 수 있다.
- 발행은 사람이 검토 패널 맨 아래 "검토 완료 후 ContentFlow에 발행"을 눌러서 한다. 자동 발행은 없다. 일괄 발행 기능도 아직 없다.
- 발행 기록(`lecture_publications`)은 수정하거나 삭제할 수 없다(append-only 트리거).

### 일괄 처리 절차 (다음 학회부터 이대로)
1. 스위치 OFF 상태에서 `/lectures/batch` → "실행기 위치 확인"이 "미국 · 실행 가능"인지 본다.
2. 강연을 고르고 "견적 계산" → 합계, 배치 상한(기본 1.5배), 처리 순서(원문 짧은 순)를 확인한다.
3. 동의를 체크하고 "강연 N건 일괄 승인 생성"을 누른다. 클라우드 세션이 DB에서 배치, 항목, 승인, run 수를 확인한다.
4. 맥미니가 스위치를 ON으로 배포한다. cron이 5분마다 동시 2건씩 처리한다.
5. 클라우드 세션이 처음 30분은 15분 간격, 이후 30~60분 간격으로 감시하고 결정이 필요한 것만 알린다.
6. 끝나면 맥미니가 스위치를 OFF로 배포한다. 강연별로 검토하고 발행한다.

### 감시 기준과 비상 정지
- 일시정지 기준: 미국 외 지점에서 실행, 견적의 1.4배를 넘는 비용 속도, 같은 단계 강제 종료 반복, 같은 오류의 재발(max_tokens 등).
- 자동 안전장치: 결정 대기 3건 연속, 배치 비용 상한 도달, 비정상 tick(예외, 지역 건너뛰기) 3회 연속이면 시스템이 스스로 일시정지한다.
- 비상 정지 순서: SQL 일시정지 → 맥미니가 스위치 OFF 배포 → Anthropic 키 비활성화.
- SQL 일시정지 (클라우드 세션이 할 수 있음, 재개는 웹 "재개" 버튼으로만):
```sql
UPDATE lecture_live_batches SET status = 'paused', paused_by = 'sql',
  paused_reason = '운영자 정지: <사유>', paused_at = now()
WHERE id = '<batch id>' AND status = 'active';
```

### 자주 쓰는 조회
```sql
-- 배치 진행
SELECT position, title, item_status, run_status, current_stage, execution_colo,
       run_cost_usd, item_estimated_cost_usd, stage_retries, last_error, decision_reason
FROM lecture_live_batch_progress WHERE batch_id = '<batch id>' ORDER BY position;
SELECT * FROM lecture_live_batch_totals WHERE batch_id = '<batch id>';
SELECT status, last_swept_at, consecutive_abnormal_ticks, last_abnormal_reason
FROM lecture_live_batches WHERE id = '<batch id>';

-- 월 비용
SELECT round(sum(cost_usd)::numeric, 4) FROM lecture_pipeline_runs
WHERE execution_mode = 'live' AND created_at >= date_trunc('month', now());

-- 열린 run과 쓰지 않은 승인 (작업 전후에 0인지 확인)
SELECT count(*) FROM lecture_pipeline_runs
WHERE execution_mode = 'live' AND status IN ('running','queued','retryable_error');
SELECT count(*) FROM lecture_external_processing_authorizations a
WHERE a.expires_at > now() AND NOT EXISTS (
  SELECT 1 FROM lecture_external_processing_authorization_uses u WHERE u.authorization_id = a.id);
```

## 4. 강연 처리 현황 (2026-10-13)

- GIN2026: 일괄 20건 + 사전 워크숍 + 시험 강연(호주 MND) = **22건 처리와 발행 완료**.
- 일괄 배치 `bc785cb8-35d9-44cb-8f89-85e1c15b68e7`: completed. 실제 비용 약 $11.65(견적 $13.37). 8번(HAS)은 옛 run `ae8bc045`(failed, superseded, $0.51)를 버리고 새 단건 run `3ba6cde9`($0.35)로 처리했다.
- 검토 메모: 3, 4, 6, 12, 15, 16, 19, 20번은 번역에서 `[?]` 표지 1개가 빠진 채 허용치로 통과했다(번호 표지 도입 전). 사용자가 1~3번을 검토했고, 나머지는 일괄 발행하기로 결정해 모두 발행했다.
- 그 밖의 강연 세션: `bc7e4d85`(영어 발표 6/28)는 예전 파이프라인으로 발행됨. 합성 테스트와 한국어 짧은 테스트 세션 몇 개는 처리 대상이 아니다.

## 5. 이번에 바뀐 것과 배운 것

맥미니 main 커밋(순서대로): `ff84f7d` 일괄 처리 → `d63646d` 미국 DO 실행기 → `2dfc00d` claim·정체·비정상 tick 보완 → `b1b6857` 실행 지점은 colo로 판정(미국 colo 47개) → `8502556`, `2c67aa2` 일괄 승인 생성 최적화(외부 호출 예산) → `2b8bdc1` 단계 출력 한도 상향 → `a703d7d` 닫힌 배치의 "다시 시도"가 배치를 다시 열도록 → `4e88704` `[?]` 번호 표지와 줄 단위 복원 → `f475b20` 기록. 현재 Worker 버전은 `a1e47815`(스위치 OFF)다.

| 문제 | 원인 | 해결 |
|---|---|---|
| cron에서 Anthropic 403 가능성 | placement는 fetch 핸들러에만 적용 | cron은 claim만 하고 실행은 `jurisdiction("us")` DO에서 |
| 실행기 위치가 "KR"로 표시 | trace의 `loc`은 요청자 국가 | 실행 지점 `colo`가 미국 목록에 있는지로 판정 |
| 일괄 승인 생성이 7건에서 끊김 | Workers 무료 요금제의 요청당 외부 호출 50회 한도 | Workers Paid 전환 + 조회를 일괄로 묶고 예산을 넘으면 나눠 보냄 |
| 교정 단계 max_tokens 초과 | 출력 한도 여유가 얇음(실측이 한도의 57~96%) | `max(예상×3, 예상+8,192)`, 모델 최대치와 run 남은 예산으로 상한 |
| 번역 `[?]` 누락 반복(8번) | 표지 밀도가 높은 조각에서 모델이 문장을 합치며 표지를 떨어뜨림 | 요청에서만 `[?1..N]` 번호 표지, 응답에서 되돌림. 2~3개까지 줄 끝 복원 + 경고 |
| 닫힌 배치에서 "다시 시도"가 안 돎 | cron은 active 배치만 처리 | 다시 시도 시 배치를 다시 엶 |

운영상 주의:
- Worker(미국 동부)와 Supabase(서울) 사이 왕복이 약 250ms라 DB 호출이 많은 요청은 느리다.
- `superseded:`로 failed 처리한 run은 강연별 지출 상한 계산에서 빠진다. 운영자(맥미니)만 쓰는 예외로 둔다.
- 실제 비용은 대체로 견적의 0.8~1.25배였다. 배치 상한 1.5배면 충분했다.

## 6. ContentFlow 전체 현황 (강연 외, DB 기준 2026-10-13)

| 영역 | 현황 |
|---|---|
| 콘텐츠 `contents` | 443건: 웹 글 367, 논문 16, 스레드 14, 기타 웹 17, 북마크 5, PDF 1, 강연 23. 모두 `is_processed`, 처리 오류 0. |
| 요약 `summaries` | 432건. 요약이 없는 콘텐츠 11건(북마크 5, 웹 6). |
| 분류 | `contents.category` 단일 값. 상위: tech.ai-tools 187, tech.dev 43, research.ai 37, opinion.column 34, research.medical 34. **분류 없음 27건(강연 21, 웹 6)**. `research_categories` 22개. |
| 다중 분류·키워드 | `content_categories`, `content_keywords` 테이블은 있지만 **0건(미사용)**. |
| 임베딩 `content_chunks` | 5,045 청크, 436개 콘텐츠, 모델 gemini-embedding-2. **미임베딩 1건: 2026-10-13 발행한 14번 JRC 강연**(오류 기록 없음). |
| 지식베이스 `knowledge_bases` | 3개 중 **2개 오류**(tech.dev, research.ai): `Gemini API error 400 "User location is not supported"`. tech.product만 ready. |
| 슬라이드 `slides` | 2건 |
| Google Drive 연동 | `gdrive_tokens` 1, `gdrive_folders` 2 |

## 7. 다음 후보 작업

강연 이외 부분의 완성도를 높이는 것이 다음 목표다. 우선순위는 사용자가 정한다. 아래는 DB에서 보인 근거가 있는 후보다.

1. **지식베이스 생성의 지역 오류**: Gemini 400 "User location is not supported". 강연에서 겪은 지역 차단과 같은 종류로 보인다. 생성이 어느 경로(fetch, cron)에서 도는지 확인하고, 강연처럼 미국 실행(placement 또는 DO) 경로로 옮긴다.
2. **분류 없는 콘텐츠 27건**: 강연 21건은 발행할 때 category가 비어 있다. 강연 발행 시 분류를 지정하는 방법(자동 제안 + 사람 확인)을 정한다.
3. **요약 없는 11건**(북마크 5, 웹 6): 재처리하거나 원인을 확인한다.
4. **미임베딩 1건**(14번 JRC 강연): 임베딩 작업이 따라오는지 확인한다.
5. **다중 분류와 키워드**(`content_categories`, `content_keywords`): 쓸지 결정하고, 쓴다면 설계한다.
6. **강연 일괄 발행 기능**: 이번에는 17건을 하나씩 눌렀다. 요청문 초안은 대화 기록에 있다(대상: 강연별 최신 awaiting_approval run, 단건 발행과 같은 코드 경로).
7. **검토 화면 개선**: `[?]` 누락 허용과 "Marker restored" 경고를 검토 패널에 눈에 띄게 표시한다.
8. **비용 방어선**: Anthropic 콘솔 월 지출 한도(예: $25)를 설정할지 결정이 아직 안 됐다.

## 8. 다음 세션 시작하기

클라우드 세션을 새로 열 때는 아래를 붙여 넣고 시작하면 된다.

```
ContentFlow 작업을 이어서 하려고 해. 저장소 ZZangbo/Contentflow의
handoff/CONTENTFLOW_HANDOFF.md를 먼저 읽고, Supabase(wlvhmuudjylaayunnkop)로
현재 상태(열린 run, 미사용 승인, 이번 달 비용)를 확인한 뒤 시작해 줘.
오늘 할 일: <할 일>
```

맥미니 세션은 `claude --continue`로 이어 가고, 맥미니의 CLAUDE.md와 메모리, `~/Desktop/contentflow-batch-설계.md`에 세부 기록이 있다.
