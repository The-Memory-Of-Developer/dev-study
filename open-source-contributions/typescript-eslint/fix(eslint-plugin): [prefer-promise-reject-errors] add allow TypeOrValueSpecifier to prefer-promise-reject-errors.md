# fix(eslint-plugin): [prefer-promise-reject-errors] add allow TypeOrValueSpecifier to prefer-promise-reject-errors

## 링크

- Issue: https://github.com/typescript-eslint/typescript-eslint/issues/12048
- PR: https://github.com/typescript-eslint/typescript-eslint/pull/12094
- Changes: https://github.com/typescript-eslint/typescript-eslint/pull/12094/changes
- [prefer-promise-reject-errors] rule 문서 수정 : https://typescript-eslint.io/rules/prefer-promise-reject-errors

## 1. 문제 현상(재현 코드)

`@typescript-eslint/prefer-promise-reject-errors`는 Promise rejection reason을 `Error` 계열로 강제하는 규칙입니다.
그런데 `only-throw-error`와 달리 `allow: TypeOrValueSpecifier[]`를 지원하지 않아, throw 경로와 reject 경로에 동일한 예외 정책을 재사용하기 어려웠습니다.

즉, 모노레포/공통 ESLint 설정에서 다음과 같은 불일치가 발생했습니다.

- `only-throw-error`: `allow`로 세밀한 예외 허용 가능
- `prefer-promise-reject-errors`: 동일 포맷 재사용 불가

## 2. 원인 분석

규칙 구현을 확인한 결과, `prefer-promise-reject-errors`에는 아래 요소가 누락되어 있었습니다.

- 옵션 타입에 `allow?: TypeOrValueSpecifier[]` 정의
- schema에 `allow` 항목(`typeOrValueSpecifiersSchema`) 반영
- defaultOptions에 `allow: []` 초기값 반영
- 타입 매칭 로직에서 `allow` 스펙 기반 예외 허용 분기

결과적으로 사용자는 throw 경로와 reject 경로를 같은 정책으로 설정할 수 없었습니다.

## 3. 해결 방향

해결 방향은 `only-throw-error`와 정책 모델을 맞추는 것이었습니다.

- `prefer-promise-reject-errors`에 `allow?: TypeOrValueSpecifier[]` 옵션을 추가
- rejection reason의 타입이 `allow` 스펙에 매칭되면 리포트하지 않도록 예외 경로 추가
- 기존 옵션(`allowThrowingAny`, `allowThrowingUnknown`, `allowEmptyReject`)과 충돌 없이 공존하도록 설계

## 4. 코드 수정

수정 파일: `packages/eslint-plugin/src/rules/prefer-promise-reject-errors.ts`

옵션 타입/스키마/기본값 확장:

```ts
type Options = [
  {
    allow?: TypeOrValueSpecifier[];
    allowEmptyReject?: boolean;
    allowThrowingAny?: boolean;
    allowThrowingUnknown?: boolean;
  },
];

schema: [
  {
    type: 'object',
    additionalProperties: false,
    properties: {
      // 기존 옵션 유지
      allowEmptyReject: { type: 'boolean' },
      allowThrowingAny: { type: 'boolean' },
      allowThrowingUnknown: { type: 'boolean' },
      // 추가: only-throw-error와 동일한 allow 포맷 지원
      allow: typeOrValueSpecifiersSchema,
    },
  },
],

defaultOptions: [
  {
    allow: [],
    allowEmptyReject: false,
    allowThrowingAny: true,
    allowThrowingUnknown: true,
  },
],
```

핵심 타입 허용 분기 추가:

```ts
if (
  // allow 스펙에 매칭되는 타입이면 reject reason으로 허용
  typeMatchesSomeSpecifier(type, options.allow, services.program)
) {
  return;
}
```

위 분기를 통해 `Promise.reject(...)`와 `new Promise((_, reject) => reject(...))` 경로 모두 동일한 정책을 적용하도록 했습니다.

## 5. 테스트 보강

수정 파일: `packages/eslint-plugin/tests/rules/prefer-promise-reject-errors.test.ts`

추가한 `valid` 테스트(allow 적용 시 통과):

```ts
// https://github.com/typescript-eslint/typescript-eslint/issues/12048
{
  code: `
    class CustomRejection {}
    Promise.reject(new CustomRejection());
  `,
  options: [{ allow: [{ from: 'file', name: 'CustomRejection' }] }],
},
{
  code: `
    class CustomRejection {}
    Promise.reject(new CustomRejection());
  `,
  options: [{ allow: ['CustomRejection'] }],
},
{
  code: `
    Promise.reject(new Date());
  `,
  options: [{ allow: [{ from: 'lib', name: 'Date' }] }],
},
{
  code: `
    new Promise((resolve, reject) => reject(new Date()));
  `,
  options: [{ allow: [{ from: 'lib', name: 'Date' }] }],
},
{
  code: `
    import { createError } from 'errors';
    Promise.reject(createError());
  `,
  options: [
    {
      allow: [{ from: 'package', name: 'ErrorLike', package: 'errors' }],
    },
  ],
},
{
  code: `
    import { createError } from 'errors';
    new Promise((resolve, reject) => reject(createError()));
  `,
  options: [
    {
      allow: [{ from: 'package', name: 'ErrorLike', package: 'errors' }],
    },
  ],
},
```

추가한 `invalid` 테스트(allow 미설정 시 리포트):

```ts
// https://github.com/typescript-eslint/typescript-eslint/issues/12048
{
  code: `
class CustomRejection {}
Promise.reject(new CustomRejection());
  `,
  errors: [{ messageId: 'rejectAnError' }],
},
{
  code: `
Promise.reject(new Date());
  `,
  errors: [{ messageId: 'rejectAnError' }],
},
{
  code: `
import { createError } from 'errors';
Promise.reject(createError());
  `,
  errors: [{ messageId: 'rejectAnError' }],
},
{
  code: `
import { createError } from 'errors';
new Promise((resolve, reject) => reject(createError()));
  `,
  errors: [{ messageId: 'rejectAnError' }],
},
```

위 테스트로 `file/string/lib/package` allow 경로와 `Promise.reject(...)` / `new Promise(...reject(...))` 경로를 모두 검증했습니다.

## 6. 결과 및 기여

- `prefer-promise-reject-errors`에서 `allow: TypeOrValueSpecifier[]`를 공식 지원하게 되었습니다.
- throw/reject 경로 간 예외 정책을 동일한 형태로 재사용할 수 있게 되어 설정 일관성이 개선되었습니다.
- 모노레포/공통 ESLint 설정에서 규칙 간 정책 관리 비용을 줄였습니다.
- `packages/eslint-plugin/docs/rules/prefer-promise-reject-errors.mdx` 문서에 `allow` 옵션 설명/타입/기본값을 추가했습니다.
- `packages/eslint-plugin/tests/schema-snapshots/prefer-promise-reject-errors.shot` 스냅샷을 갱신해 스키마 변경을 테스트 자산과 동기화했습니다.

## 7. 학습 포인트

- `only-throw-error`와 `prefer-promise-reject-errors`는 적용 경로는 다르지만, 정책 모델을 통일해야 사용자 경험이 좋아진다는 점을 확인했습니다.
- 규칙 기능 추가 시 구현 코드만이 아니라 schema/defaultOptions/test/docs/snapshot까지 함께 갱신해야 안정적으로 머지할 수 있다는 점을 체감했습니다.
- 중간 실패(스냅샷/실행 옵션)는 기능 설계 문제와 분리해 빠르게 정리하는 것이 생산성에 중요하다는 점을 학습했습니다.
