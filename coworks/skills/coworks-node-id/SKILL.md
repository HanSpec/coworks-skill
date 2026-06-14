---
name: coworks-node-id
description: "Use when the user gives a coworks node id to work on. Triggers on /coworks:coworks-node-id <id>, 'coworks id NNN 구현/작업', 'node id NNN 진행', or when a coworks REQUIREMENT node needs to be fetched, analyzed, implemented, registered as tasks, and marked done end-to-end."
---

# coworks node id — 요구사항 조회→구현→완료 자동화

> coworks 서버의 REQUIREMENT node id 하나를 받아, 조회 → 관련 요구사항·기존 task 파악 →
> 구현 → 관련 컴포넌트 task 등록 → 완료(DONE) 처리까지 **끝까지 자동**으로 진행한다.
> 사용자가 매번 node 내용·task를 직접 알려주던 반복 작업을 대체한다.

인자: node id (예: `/coworks:coworks-node-id 226`). 없으면 사용자에게 묻는다.

---

## 사전 조건 (호출 정보)

이 스킬은 coworks HTTP API를 호출한다. 다음 환경변수 규약을 사용한다.

- `HANSPEC_COWORKS_BASE_URL` — coworks 서버 주소. 로컬 작업 시 보통 `http://localhost:3000`,
  배포는 coworks 연동 가이드 참조.
- `HANSPEC_COWORKS_ACCESSTOKEN` — coworks `/me`에서 발급한 액세스 토큰(7일 유효).

호출 측 프로젝트의 `.env`에 위 두 변수가 있으면 그대로 쓴다. 둘 다 없으면 사용자에게
"coworks 토큰/BASE_URL 환경변수가 필요합니다(coworks `/me`에서 발급)"라고 알리고 중단한다.

> **이 스킬은 오직 coworks HTTP API만 사용한다.** 조회·task 등록·완료(DONE) 처리 모두
> 토큰 API로 한다. **coworks DB나 coworks 레포 소스에 직접 접근하지 않는다** — 호출 측
> 프로젝트(외부)는 coworks 내부에 접근 권한이 없는 게 정상이며, 그럴 필요도 없다.

---

## 실행 절차

### Step 1: node 조회

```bash
curl -s "$HANSPEC_COWORKS_BASE_URL/api/nodes/<id>" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN"
curl -s "$HANSPEC_COWORKS_BASE_URL/api/nodes/<id>/tasks" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN"
```

응답에서 파악할 것:
- `level` — REQUIREMENT가 아니면(MODULE/FEATURE) 작업 대상이 아니므로 사용자에게 확인.
- `module` / `feature` / `requirement` — 작업의 맥락(어떤 화면·기능인지).
- `requirement.description` — **구현해야 할 내용**. 비어 있으면 module/feature 이름과
  task 설명에서 의도를 추론한다.
- `requirement.status` — 이미 DONE이면 재작업 여부를 사용자에게 확인.
- **`related`** — 관련 요구사항 배열. **반드시 확인한다.** 같은 패턴/공통 기능을 요구하는
  경우가 많다(예: 한쪽에서 만든 방식을 그대로 이어받아 구현). 관련 요구사항의
  `endpoint`·`status`로 이미 구현된 선례가 있는지 본다.
- tasks 목록 — 각 task의 `description`·`progress`. progress 0인 task가 이번에 구현할 대상.

### Step 2: 관련 요구사항·선례 분석

`related`에 연결된 요구사항이 있으면 그 node도 조회해(필요 시 `/api/nodes/:id`)
**이미 구현된 동일·유사 기능의 코드 패턴을 그대로 재사용**한다. 일관성이 핵심이다.
(이 프로젝트는 "관련 요구사항끼리 같은 방식으로 구현"하는 흐름이 잦다.)

### Step 3: 구현

**현재 작업 중인 프로젝트**에서 요구사항을 구현한다. 요구사항 내용이 어느 코드베이스의 것인지는
node의 module/feature/description 맥락으로 판단한다(요구사항이 다른 프로젝트의 것이면 그 점을
사용자에게 확인).

- **검증**: 해당 프로젝트의 빌드/테스트로 확인하고, 가능하면 실제 동작까지 본다.
- 커밋·푸시는 해당 프로젝트의 규칙을 따른다. 커밋 메시지에 요구사항 번호(`#<id>`)를 남긴다.

### Step 4: 관련 컴포넌트를 task로 등록

구현한 **관련 컴포넌트/파일을 task로 등록**한다(사용자가 자주 요청하는 단계).

- 이미 있는 task(progress 0)는 `PATCH /api/tasks/:id`로 구현 완료 반영:
  `name`(컴포넌트명)·`endpoint`(파일 경로)·`progress:100`·필요 시 `description` 보강.
- 새로 만든 컴포넌트는 `POST /api/tasks`로 추가: `{ nodeId, name, endpoint, description, progress:100 }`.

```bash
# 기존 task 수정
curl -s -X PATCH "$HANSPEC_COWORKS_BASE_URL/api/tasks/<taskId>" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN" -H "Content-Type: application/json" \
  -d '{"name":"<컴포넌트명>","endpoint":"<파일경로>","progress":100}'

# 새 task 추가
curl -s -X POST "$HANSPEC_COWORKS_BASE_URL/api/tasks" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN" -H "Content-Type: application/json" \
  -d '{"nodeId":<id>,"name":"<컴포넌트명>","endpoint":"<파일경로>","description":"<설명>","progress":100}'
```

endpoint에는 파일 경로를 적는다(여러 개면 콤마로 구분). `{{ENV}}` 치환도 지원된다.

### Step 5: 완료(DONE) 처리

**`PATCH /api/nodes/:id`에 `status`를 보내 완료 처리한다.** (DB 직접 접근 불필요.)

```bash
curl -s -X PATCH "$HANSPEC_COWORKS_BASE_URL/api/nodes/<id>" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN" -H "Content-Type: application/json" \
  -d '{"status":"DONE"}'
```

- `status`는 REQUIREMENT 노드만 변경 가능. 비DONE→DONE이면 서버가 `completedAt` 기록과
  예약된 완료 알림 발송까지 처리한다(웹 UI와 동일 규칙). 호출 측이 신경 쓸 게 없다.
- 응답 `{ ok:true, node:{ id, status:"DONE", completedAt } }`로 결과를 확인한다.
- `422`(REQUIREMENT 아님)/`403`(권한)/`401`(토큰)이면 그대로 사용자에게 보고한다.

### Step 6: 커밋·푸시 + 결과 보고

- 논리적 작업 단위로 커밋한다(coworks 규칙: 매 작업마다 커밋). 커밋 전 비공개 프로젝트
  언급 재확인. 커밋 메시지에 요구사항 번호(`#<id>`)를 남긴다.
- 사용자에게 보고: 구현 내용(파일), 등록한 task 목록(id·name·endpoint), DONE 처리 결과,
  검증 결과. 관련 요구사항을 참고했으면 그 점도 명시.

---

## 주의사항

- **이 스킬은 coworks HTTP API만 쓴다.** coworks DB·소스에 직접 접근하지 않는다(외부
  프로젝트에서도 동작해야 한다). 조회·task 등록·완료 처리 모두 토큰 API로 한다.
- **REQUIREMENT 레벨**만 작업 대상이다. MODULE/FEATURE id가 오면 의도를 사용자에게 확인.
- `related`(관련 요구사항)는 항상 확인한다 — 같은 기능을 다른 node가 이미 구현한 경우 그
  패턴을 재사용해 일관성을 유지한다.
- 비공개 프로젝트의 이름·경로를 공개 산출물(공개 레포 코드·문서·커밋)에 남기지 않는다.
- 토큰이 만료(`401`)면 coworks `/me`에서 재발급하도록 안내한다.
- 되돌리기 어려운 외부 영향(공개 푸시 등) 외에는 사용자 확인 없이 끝까지 진행한다.
