# fix(eslint-plugin): [array-type] ignore Array and ReadonlyArray without type arguments

## 링크

- Issue: https://github.com/typescript-eslint/typescript-eslint/issues/11964
- PR: https://github.com/typescript-eslint/typescript-eslint/pull/11971
- Changes: https://github.com/typescript-eslint/typescript-eslint/pull/11971/changes

## 1. 문제 현상(재현 코드)

```ts
type Generic<Array extends unknown[]> = { array: Array };

// secondary example
declare module '2' {
  type Array<Y> = Y;
  const y: Array<2>;
}
```

위 코드에서 `Array`는 전역 내장 타입이 아니라, 로컬 타입 파라미터/로컬 타입 별칭입니다.
하지만 기존 `@typescript-eslint/array-type` 규칙은 이를 전역 `Array`로 오인해 잘못 리포트했습니다.

- 실제 오탐 메시지: `Array type using 'Array' is forbidden. Use 'any[]' instead.`

## 2. 원인 분석

원인을 확인한 결과, `array-type` 규칙의 판정 로직이 `Array`/`ReadonlyArray` 식별자를 이름 중심으로 처리하고 있었습니다.

- 식별자가 현재 스코프에서 shadowing(재정의)된 타입인지 확인하는 단계가 부족했습니다.
- `Array`/`ReadonlyArray` 참조가 전역 내장 타입인지, 로컬 선언인지 구분이 불충분했습니다.
- 그 결과 타입 파라미터(`Array`)나 로컬 타입 alias(`type Array<...>`)까지 전역 타입처럼 처리되어 false positive가 발생했습니다.

## 3. 해결 방향

해결 방향은 규칙이 **전역 내장 `Array`/`ReadonlyArray`에만** 개입하도록 판정 범위를 좁히는 것이었습니다.  
이를 위해 식별자 이름만으로 처리하지 않고, 해당 식별자가 현재/상위 스코프에서 이미 선언되어 shadowing된 이름인지 먼저 확인하도록 흐름을 바꿨습니다.  
이 검사에서 로컬 선언으로 확인되면 규칙 리포트를 중단해 false positive를 방지하고, 전역 내장 타입으로 안전하게 해석되는 경우에만 기존 스타일 강제 로직이 동작하도록 정리했습니다.  
또한 타입 인수가 없는 `Array`/`ReadonlyArray` 표기(`let x: Array`)에 대해서는 자동 수정으로 `any[]`를 강제하지 않고 그대로 통과시키도록 바꿔, 타입 정보가 비어 있는 선언을 과하게 치환하던 동작도 함께 정리했습니다.

## 4. 코드 수정

수정 파일: `packages/eslint-plugin/src/rules/array-type.ts`

```ts
// TSTypeReference
for (let scope = context.sourceCode.getScope(node); scope.upper; scope = scope.upper) {
  if (scope.set.has(node.typeName.name)) {
    return;
  }
}
```

이 변경으로 로컬 제네릭/타입 별칭과 전역 내장 타입을 구분해, 오탐을 줄이면서 기존 규칙 의도는 유지했습니다.

## 5. 테스트 보강

수정 파일: `packages/eslint-plugin/tests/rules/array-type.test.ts`

테스트는 "shadowed 식별자 오탐 방지"와 "타입 인수 없는 Array 처리 변경"을 함께 검증하도록 보강했습니다.

추가한 `valid` 테스트 전체:

```ts
    {
      code: 'type Generic<Array extends unknown[]> = { array: Array };',
      options: [{ default: 'array' }],
    },
    {
      code: 'type Generic<ReadonlyArray> = { array: ReadonlyArray };',
      options: [{ default: 'array' }],
    },
    {
      code: `
        declare module '2' {
          type Array<Y> = Y;
          const y: Array<2>;
        }
      `,
      options: [{ default: 'generic' }],
    },
    {
      code: 'let x: Array;',
      options: [{ default: 'array' }],
    },
    {
      code: 'let x: Array;',
      options: [{ default: 'array-simple' }],
    },
    {
      code: "let z: Array = [3, '4'];",
      options: [{ default: 'array' }],
    },
    {
      code: "let z: Array = [3, '4'];",
      options: [{ default: 'array-simple' }],
    },
```

위 테스트 변경으로 shadowed 식별자 오탐이 사라졌는지, 그리고 타입 인수 없는 `Array`를 더 이상 자동 치환하지 않는 새 동작이 의도대로 적용되는지 확인했습니다.

## 6. 결과 및 기여

- `array-type` 규칙에서 `Array`/`ReadonlyArray` shadowing 케이스의 false positive를 제거했습니다.
- 식별자 이름만으로 판단하던 경로를 스코프 기반으로 보강해 규칙 신뢰도를 개선했습니다.
- 로컬 타입 선언과 전역 내장 타입을 구분하는 회귀 테스트를 추가해 안정성을 높였습니다.

## 7. 학습 포인트

- 린트 규칙에서 식별자 이름 매칭만으로는 정확도가 부족하고, 스코프/심볼 해석이 함께 필요하다는 점을 확인했습니다.
- 스타일 규칙도 타입 시스템 문맥(제네릭 파라미터, 지역 타입 별칭)을 고려하지 않으면 false positive가 쉽게 발생한다는 점을 체감했습니다.
- 이슈 재현 코드 하나를 기준으로 최소 수정 + 회귀 테스트를 붙이는 방식이 리뷰/머지 과정에서 가장 설득력이 높았습니다.
