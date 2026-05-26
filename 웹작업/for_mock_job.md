# 모의고사(mock) 연동 작업 참조

새 API·도메인 로직을 넣을 때 **`features/`** 아래에 모듈을 두거나, 라우트 전용이면 **`app/(…)/…/features/`** 형태로 분리하는 것을 권장합니다. (`features/_shared`의 `api.utils`, 타입 등을 재사용하면 됩니다.)

---

## 1. 참조 모델 (TypeScript)

### 1.1 `TopikMockTest`

- **파일:** `features/question/mock/type.ts`
- **용도:** 모의고사(세트) 메타 정보

| 필드                 | 타입       | 비고       |
| ------------------ | -------- | -------- |
| `_id`              | `string` |          |
| `namePatternCode`  | `string` | l10n 코드  |
| `namePatternIndex` | `number` |          |
| `timeRestriction`  | `number` |          |
| `questionCount`    | `number` |          |
| `totalScore`       | `number` |          |
| `audioId`          | `string` | optional |



### 1.2 `TopikQuestion`

- **파일:** `features/question/question/type.ts`
- **용도:** TOPIK 개별 문항

핵심 필드만 요약:

| 필드 | 타입 | 비고 |
|------|------|------|
| `_id` | `string` | |
| `question` | `string` | |
| `isGeneral` | `boolean` | 일반 문항 `true`, 모의고사 소속 `false` |
| `name` / `namePatternCode` / `namePatternIndex` | optional | 표기·l10n |
| `mainContent`, `options`, `correctAnswer`, `correctAnswerIndex` | optional | |
| `score` | `number` | |
| `level` | `number` | 1~6 |
| `answerType` | `'text' \| 'image' \| 'writing'` | |
| `type` | `TopikQuestionType` | `'listening' \| 'reading' \| 'writing'` |
| `timeRestriction` | `number` | |
| `imageId`, `audioId` | optional | |
| `additionalQuestions` | `AdditionalQuestion[]` | optional |
| `explanations` | `ExplanationByLang` | optional |
| 기타 | | `category`, `phrase`, `videoStartPoint`, 텍스트 해설 필드 등 |

전체 정의는 해당 `type.ts`를 기준으로 합니다.

### 1.3 `UserProfile`

- **백엔드 모델:** `UserTopikProfile` (`src/models/topik/UserTopikProfile.ts`)
- **용도:** 유저별 구독·크레딧 정보. `User` 도큐먼트와 `userId`로 연결되며, 프로필 API 응답에 `User`의 이메일·닉네임 등을 병합해 반환.
- **파일(프론트):** `features/user/type.ts` 에 아래 타입을 추가할 것.

**DB 필드 (`UserTopikProfile`)**

| 필드                       | 타입                | 비고                                                                   |
| ------------------------ | ----------------- | -------------------------------------------------------------------- |
| `userId`                 | `string`          | User._id 문자열, unique (isDeleted=false 기준)                            |
| `credits`                | `number`          | 보유 크레딧, 기본값 0                                                        |
| `subscription`           | `string`          | 구독 플랜 코드. `'none'` \| `'1d'` \| `'1m'` \| `'3m'` \| `'6m'` \| `'1y'` |
| `subscriptionExpiration` | `Date` (optional) | 구독 만료일                                                               |
| `payedAmount`            | `number`          | 누적 결제 금액, 기본값 0                                                      |

**API 응답에 병합되는 `User` 필드**

| 필드 | 타입 | 비고 |
|------|------|------|
| `email` | `string` (optional) | |
| `username` | `string` (optional) | |
| `displayName` | `string` (optional) | |
| `profilePicture` | `string` (optional) | Wasabi Presigned URL |
| `isEmailVerified` | `boolean` | |
| `loginMethod` | `'google' \| 'apple' \| 'kakao' \| 'email'` | googleId/appleId/kakaoId 유무로 판별 |
| `timezone` | `string` (optional) | |

**프론트 타입 예시**

```ts
// features/user/type.ts
export interface UserTopikProfile {
  _id: string;
  userId: string;
  credits: number;
  subscription: string;
  subscriptionExpiration?: string; // ISO 8601
  payedAmount: number;
  // User 병합 필드
  email?: string;
  username?: string;
  displayName?: string;
  profilePicture?: string;
  isEmailVerified: boolean;
  loginMethod: 'google' | 'apple' | 'kakao' | 'email';
  timezone?: string;
}
```

---

## 2. 로그인 (클라이언트 → Next API)

- **UI:** `app/(app)/admin/login/page.tsx`
- **클라이언트 호출**
  - **URL:** `POST /api/auth/login` (상대 경로, 같은 오리진)
  - **헤더:** `Content-Type: application/json`
  - **바디:** `{ email: string, password: string }`
- **성공 시:** `res.ok` 이면 세션 쿠키 등이 브라우저에 설정된 뒤 리다이렉트하는 흐름(라우팅은 UI에서 처리).
- **실패 시:** JSON에서 `message` 또는 `error`를 메시지로 사용하는 패턴.
- **참고:** `lorotemp@e4net.net` 등 **예외 리다이렉트**는 **이 문서 범위에서 제외** (로그인 API 계약만 참고).

### 2.1 Next Route Handler (실제 백엔드 위임)

- **파일:** `app/(app)/api/auth/login/route.ts`
- **업스트림:** `POST {API_HOST}/api/v1/auth/login` (환경 변수 `API_HOST` 필수)
- **요청 바디:** `{ email, password }` (JSON)
- **성공 시(요약):** 응답 JSON의 `data.tokens.accessToken` (및 `refreshToken`) 를 HttpOnly 쿠키 `access_token` / `refresh_token` 으로 설정하는 흐름.  
- 상세·에러 shape는 위 route 파일을 기준.

---

## 3. 모의고사 목록

- **서비스:** `features/question/mock/service.ts`
- **현재 함수:** `getTopikMockTests(page, limit)` → `getAndUnwrapJSON("api/v1/topik-mock-tests", { page, limit })`
- **인프라:** `features/_shared/api.utils.ts` 의 `getAndUnwrapJSON` → `infra/api/client`의 `getJSON` 사용. 백엔드는 통상 **`APIResponse<T>`** (`status: 'success'`, `data`, …) 래핑.

### 3.1 쿼리 파라미터 확장 필요(API에서는 대응되어 있음)

- **`type`:** `'listening' | 'reading'`
- **의도:** 읽기 / 듣기 모의고사만 필터.
- **호출 예:** `getAndUnwrapJSON("api/v1/topik-mock-tests", { page, limit, type })`  
- 구현 시 `getTopikMockTests` 시그니처에 `type?: 'listening' | 'reading'` 추가하면 됨.

---

## 4. 모의고사에 연결된 문제 일괄 조회

- **서비스:** `features/question/question/service.ts`
- **함수:** `getTopikQuestionsByMockTestId(mockTestId: string)`
- **코드:**
  ```ts
  getByIdAndUnwrapJSON("api/v1/topik-questions/mock", mockTestId)
  ```

### 4.1 백엔드 요청 형태 (`infra/api/client`)

- `getJSONById`는 리소스 경로 뒤에 ID를 붙입니다.  
  **실제 요청 경로:** `GET …/api/v1/topik-questions/mock/{mockTestId}`  
  (`{PROXY_BASE_PATH}` 프리픽스 + `api/v1/topik-questions/mock` + `/` + `encodeURIComponent(mockTestId)`)
- **Method:** `GET`
- **Body:** 없음
- **Response (래핑):** `getByIdAndUnwrapJSON` → 표준 **`APIResponse<T>`** unwrap 후 **`data`가 `TopikQuestion[]`** 로 가정되어 있음 (`Promise<TopikQuestion[] | null>`).
- **문항 모델:** §1.2 `TopikQuestion` 참고.

백엔드가 쿼리 파라미터·인증·에러 코드를 어떻게 주는지는 **별도 명세**(또는 `APIResponse`의 `error` shape)와 맞출 것.

---

## 5. 유저 정보 조회 (프로필)

- **백엔드 엔드포인트:** `src/endpoints/topik/userTopikProfile.endpoints.ts`
- **응답 타입:** `UserTopikProfile` (§1.3 참조)

### 5.1 프로필 조회

- **Method / Path:** `GET /api/v1/topik/user/profile`
- **Auth:** Bearer 토큰 필수 (authenticatedEndpointsFactory)
- **Request:** 없음
- **Response:**

```json
{
  "_id": "string",
  "userId": "string",
  "credits": 0,
  "subscription": "none",
  "subscriptionExpiration": "2025-01-01T00:00:00.000Z",
  "payedAmount": 0,
  "email": "string",
  "username": "string",
  "displayName": "string",
  "profilePicture": "https://presigned-url...",
  "isEmailVerified": true,
  "loginMethod": "email",
  "timezone": "Asia/Seoul"
}
```

- `subscriptionType`: `'1d'` | `'1m'` | `'3m'` | `'6m'` | `'1y'`
- `amount`: 결제 금액(정수)
- **Response:** `{ success: true, message: "Subscription updated", data: UserTopikProfile }`

### 5.2 크레딧 추가

- **Method / Path:** `POST /api/v1/topik/user/credit`
- **Auth:** Bearer 토큰 필수
- **Request Body:**

```json
{
  "credits": 100,
  "amount": 1000
}
```

- **Response:** `{ success: true, message: "Credits added", data: UserTopikProfile }`

---

## 6. 구매한 모의고사 목록(백엔드 API 아직 미작성, 프로토타입 단계에서는 사용하지 않아도 문제 없음)

```text
(추가 예정)
- Method / Path:
- Response: (구매/라이선스와 연동된 `TopikMockTest[]` 또는 ID 목록 등)
```

---

## 7. 관련 파일 빠른 링크

| 구분 | 경로 |
|------|------|
| 모의고사 타입 / 서비스 | `features/question/mock/type.ts`, `service.ts` |
| 문항 타입 / 서비스 | `features/question/question/type.ts`, `service.ts` |
| API 래퍼 | `features/_shared/api.utils.ts` |
| 로그인 페이지 | `app/(app)/admin/login/page.tsx` |
| 로그인 API route | `app/(app)/api/auth/login/route.ts` |
| 유저(관리자 리스트용 `User`) | `features/user/type.ts` (프로필 API와는 별개로 취급) |

