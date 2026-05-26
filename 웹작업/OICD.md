# OIDC 작업 요약

  

- OIDC 로그인(NextAuth + `loro-topik` provider) 테스트 플로우를 구성했다.

- 로그인 후 `sso-finalize`에서 백엔드 `/api/v1/auth/sso-login` 호출로 loro 토큰 쿠키(`access_token`, `refresh_token`, `ui_role`)를 세팅하도록 연결했다.

- 백엔드 미구현/검증 실패 구간을 확인했고, 백엔드 API 명세(`OIDC_SSO_BE_API.md`)를 정리했다.

  

## `app/(popup)/layout.tsx` 변경사항 (소규모)

  

- 기존처럼 `sessionStorage.user_session`만으로 로그인 상태를 판단하지 않도록 변경

- access_token 쿠키가 있고 세션이 없을 때 `GET /api/proxy/api/v1/topik/user/profile`를 호출해, HttpOnly 쿠키 기반 인증이 유효하면 `sessionStorage`를 복구

  - 저장 키: `user_session`, `current_user_id`

- 라우팅 가드 조정:

  - 미로그인: `/mock-exam/login` 이동

  - 로그인 + 튜토리얼 미완료: `/mock-exam/tutorial` 이동

  - 로그인 + 튜토리얼 완료 상태에서 `/mock-exam/login` 또는 `/mock-exam/tutorial` 접근 시 `/mock-exam/welcome` 이동

  

## 추가 메모

  

- `app/(popup)/mock-exam/login/page.tsx`에 임시 로그인 버튼(테스트 OIDC 프로바이더)과 로그아웃 버튼을 추가해둔 상태이며, 필요 시 원하는 형태로 수정해도 괜찮음.

- PC 모의고사에서 수정되어야 할 항목은 별도 시트에서 정리/기입 중이므로 함께 참조:

  - [로로토픽 모의고사 검수 현황 (Google Sheets)](https://docs.google.com/spreadsheets/d/1-TgAUl3sCRrl1W1qdOCws2f0_V_MxJJUHOLI-yVFXo8/edit?gid=572269734#gid=572269734)