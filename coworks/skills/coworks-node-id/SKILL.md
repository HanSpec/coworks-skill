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

> **상태 변경(DONE) 주의**: 현재 coworks에는 node status를 바꾸는 토큰 API가 없다.
> 완료 처리는 coworks 레포에서 DB를 직접 갱신해야 한다(아래 Step 5). 따라서 이 스킬의
> 완료 단계는 **coworks 레포(`hanspec/coworks`)에 접근 가능한 환경**에서만 동작한다.

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

coworks 레포에서 요구사항을 구현한다. 산출물 위치는 `role`/도메인에 따른다(루트 CLAUDE.md
"Role별 산출물 위치" 참조). 프론트엔드면 `src/app/...`·`src/components/...` 등.

- **검증**: `pnpm run build`로 컴파일 확인. 가능하면 로컬 dev 서버로 실제 동작까지 확인한다
  (세션 쿠키가 필요한 페이지는 `SESSION_SECRET`으로 검증용 쿠키 생성 — 아래 참고).
- **비공개 프로젝트 언급 금지**: coworks는 오픈소스다. 코드·문서·커밋 메시지에 호출 측
  비공개 프로젝트 이름/경로를 남기지 않는다. 커밋 전 `git grep -i <비공개프로젝트명>`으로 확인.

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

토큰 API가 없으므로 coworks 레포에서 DB를 직접 갱신한다(웹 액션 `updateNodeStatus`와 동일 규칙).

1. 예약된 완료 알림 확인: `completeNotification`에 trigger/target이 이 node인 행이 있으면
   웹과 동일하게 발송 처리도 필요(없으면 status만 갱신해도 됨).
2. 비DONE→DONE 전환 시 `completedAt: new Date()` 함께 기록(DONE 해제는 `completedAt: null`).

coworks 레포 루트에서 임시 tsx 스크립트로 갱신하고, 쓰고 나서 삭제한다:

```ts
// tmp-done.mts  (실행: pnpm dlx tsx --env-file=.env tmp-done.mts; 끝나면 rm)
import { PrismaClient } from "./src/generated/prisma/client.ts";
import { PrismaPg } from "@prisma/adapter-pg";
const prisma = new PrismaClient({
  adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL! }),
});
const noti = await prisma.completeNotification.findMany({
  where: { OR: [{ triggerNodeId: <id> }, { targetNodeId: <id> }] },
});
console.log("notifications:", JSON.stringify(noti));
const updated = await prisma.node.update({
  where: { id: <id> },
  data: { status: "DONE", completedAt: new Date() },
  select: { id: true, status: true, completedAt: true },
});
console.log("updated:", JSON.stringify(updated));
await prisma.$disconnect();
```

### Step 6: 커밋·푸시 + 결과 보고

- 논리적 작업 단위로 커밋한다(coworks 규칙: 매 작업마다 커밋). 커밋 전 비공개 프로젝트
  언급 재확인. 커밋 메시지에 요구사항 번호(`#<id>`)를 남긴다.
- 사용자에게 보고: 구현 내용(파일), 등록한 task 목록(id·name·endpoint), DONE 처리 결과,
  검증 결과(빌드/실측). 관련 요구사항을 참고했으면 그 점도 명시.

---

## 참고: 세션 쿠키가 필요한 페이지 실측

로그인 필요 페이지는 coworks `SESSION_SECRET`으로 검증용 쿠키를 만들어 curl로 확인한다.

```bash
COOKIE=$(node -e '
const { createHmac } = require("node:crypto");
const payload = `<memberId>.${Date.now()}`;
const sig = createHmac("sha256", process.env.SESSION_SECRET).update(payload).digest("base64url");
console.log(`${payload}.${sig}`);
')
curl -s -b "coworks_session=$COOKIE" "$HANSPEC_COWORKS_BASE_URL/project/<pid>/..."
```

---

## 주의사항

- **REQUIREMENT 레벨**만 작업 대상이다. MODULE/FEATURE id가 오면 의도를 사용자에게 확인.
- `related`(관련 요구사항)는 항상 확인한다 — 같은 기능을 다른 node가 이미 구현한 경우 그
  패턴을 재사용해 일관성을 유지한다.
- coworks는 **오픈소스**다. 비공개 협업 프로젝트의 이름·경로를 어떤 산출물에도 남기지 않는다.
- 완료 처리(Step 5)는 coworks 레포 DB 접근이 가능한 환경에서만 동작한다. 불가하면 task
  등록까지만 하고, status는 사용자가 웹 UI에서 바꾸도록 안내한다.
- 되돌리기 어려운 작업(DB 변경 등) 외에는 사용자 확인 없이 끝까지 진행한다.
