# coworks skill marketplace

coworks(HanSpec 협업 워크스페이스)와 상호작용하는 Claude Code 스킬을 배포하는 공개 마켓플레이스.

## 포함 스킬

| 스킬 | 호출 | 설명 |
|------|------|------|
| `coworks-node-id` | `/coworks:coworks-node-id <id>` | coworks REQUIREMENT node id 하나로 조회→관련 요구사항/기존 task 파악→구현→관련 컴포넌트 task 등록→완료(DONE) 처리까지 자동 진행 |

## 설치

```
/plugin marketplace add HanSpec/coworks-skill
/plugin install coworks@coworks-marketplace
```

## 사전 조건 (환경변수)

coworks HTTP API를 호출하므로 호출 측 프로젝트에 다음 환경변수가 필요하다.

- `HANSPEC_COWORKS_BASE_URL` — coworks 서버 주소 (로컬은 `http://localhost:3000`)
- `HANSPEC_COWORKS_ACCESSTOKEN` — coworks `/me`에서 발급한 액세스 토큰 (7일 유효)

자세한 흐름은 `coworks/skills/coworks-node-id/SKILL.md` 참조.
