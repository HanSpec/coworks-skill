# coworks slide update — 슬라이드 페이지 본문 업데이트 자동화

> coworks 슬라이드 페이지 id를 받아, 현재 버전 조회 → 본문(wiremd) 생성 → DB 저장까지
> **끝까지 자동**으로 진행한다. 페이지 기획이 끝난 뒤 슬라이드 본문을 채워 넣는 반복 작업을 대체한다.

인자: `pageId` (예: `/coworks:coworks-slide-update 3`). 없으면 사용자에게 묻는다.

---

## 사전 조건 (호출 정보)

이 스킬은 coworks HTTP API를 호출한다. 다음 환경변수 규약을 사용한다.

- `HANSPEC_COWORKS_BASE_URL` — coworks 서버 주소. 로컬 작업 시 보통 `http://localhost:3000`.
- `HANSPEC_COWORKS_ACCESSTOKEN` — coworks `/me`에서 발급한 액세스 토큰(7일 유효).

둘 다 없으면 사용자에게 "coworks 토큰/BASE_URL 환경변수가 필요합니다(coworks `/me`에서 발급)"라고 알리고 중단한다.

> **이 스킬은 오직 coworks HTTP API만 사용한다.** coworks DB·소스에 직접 접근하지 않는다.

---

## 실행 절차

### Step 1: 페이지 조회

```bash
curl -s "$HANSPEC_COWORKS_BASE_URL/api/slides/pages/<pageId>" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN"
```

응답에서 파악할 것:
- `page.title` — 페이지 제목 (본문 작성의 맥락).
- `page.projectId` — 소속 프로젝트.
- `versions` — 버전 배열(version 내림차순). 각 버전의 `id`·`version`·`content`.
  - 최신 버전(`versions[0]`)의 `content`가 비어 있으면 → **해당 버전에 바로 작성**.
  - 최신 버전에 내용이 있으면 → **새 버전을 먼저 생성**하고(Step 2) 거기에 작성.

### Step 2: (필요 시) 새 버전 생성

최신 버전에 내용이 이미 있을 때만 실행한다.

```bash
curl -s -X POST "$HANSPEC_COWORKS_BASE_URL/api/slides/pages/<pageId>/versions" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN"
```

응답 `{ ok: true, slide: { id, version, content } }`에서 `slide.id`를 다음 단계에 사용한다.

### Step 3: 본문(wiremd) 작성

**현재 작업 중인 프로젝트·문맥**에서 페이지 제목과 목적에 맞는 wiremd 본문을 생성한다.

wiremd 문법 참고 (coworks `/api/slides/render` 엔드포인트로 미리보기 가능):
- `# 제목`, `## 소제목` — 섹션 구분.
- `[box]...[/box]` — 와이어프레임 박스.
- `(1)`, `(2)` — 번호 마커 (코멘트 연결 포인트).
- 일반 마크다운 텍스트, 목록, 코드 블록 사용 가능.

사용자가 작성할 내용을 직접 제공한 경우에는 그 내용을 그대로 사용한다.

### Step 4: 본문 저장

Step 1에서 최신 버전이 비어 있으면 그 버전의 `id`를, Step 2를 실행했으면 새 버전의 `id`를 사용한다.

```bash
curl -s -X PATCH "$HANSPEC_COWORKS_BASE_URL/api/slides/<slideId>" \
  -H "Authorization: Bearer $HANSPEC_COWORKS_ACCESSTOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"<wiremd 본문>"}'
```

응답 `{ ok: true, slide: { id, version, content } }`로 저장 결과를 확인한다.

### Step 5: 결과 보고

사용자에게 보고:
- 저장된 페이지 제목·페이지 id.
- 저장된 슬라이드 id·버전 번호.
- 본문 첫 몇 줄 미리보기.
- 확인할 수 있는 URL: `<BASE_URL>/project/<projectId>/slides/<pageId>`.

---

## 주의사항

- **이 스킬은 coworks HTTP API만 쓴다.** coworks DB·소스에 직접 접근하지 않는다.
- 최신 버전에 내용이 있으면 **반드시 새 버전을 만든 뒤** 작성한다(기존 버전 덮어쓰기 금지).
- 최신 버전이 비어 있으면(신규 페이지 첫 작성) 새 버전 없이 바로 작성한다.
- 토큰이 만료(`401`)면 coworks `/me`에서 재발급하도록 안내한다.
- 되돌리기 어려운 외부 영향 외에는 사용자 확인 없이 끝까지 진행한다.
