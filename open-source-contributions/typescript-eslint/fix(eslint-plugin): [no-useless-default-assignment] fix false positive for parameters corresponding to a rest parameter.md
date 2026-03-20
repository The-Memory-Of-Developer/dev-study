# fix(eslint-plugin): [no-useless-default-assignment] fix false positive for parameters corresponding to a rest parameter

## 링크

- Issue: https://github.com/typescript-eslint/typescript-eslint/issues/11911
- PR: https://github.com/typescript-eslint/typescript-eslint/pull/11916
- Changes: https://github.com/typescript-eslint/typescript-eslint/pull/11916/changes

## 1. 문제 현상(재현 코드)

이슈에서 보고된 최소 재현 코드는 아래와 같습니다.

```ts
const run = (cb: (...args: unknown[]) => void) => cb();
run((p: boolean = true) => null);
```

`@typescript-eslint/no-useless-default-assignment` 규칙은 위 코드를 `useless default`로 판단했지만, 기대 동작은 경고가 없어야 하는 케이스였습니다.

## 2. 원인 분석

원인을 확인하기 위해 `packages/eslint-plugin/src/rules/no-useless-default-assignment.ts`의 파라미터 판정 로직을 확인했습니다.

- 규칙은 컨텍스트 시그니처의 파라미터(`signatures[0].getParameters()`)를 인덱스로 매칭한 뒤,
- 주로 `ts.SymbolFlags.Optional` 여부를 기준으로 기본값이 불필요한지 판단하고 있었습니다.

문제는 rest parameter(`...args`)였습니다.
rest parameter는 호출 시 인자가 없어도 동작할 수 있지만, 심볼 플래그만 보면 Optional로 충분히 표현되지 않는 경우가 있습니다.
이 때문에 인라인 콜백 파라미터가 rest parameter에 매핑될 때, 규칙이 이를 필수 인자로 오인해 false positive를 발생시켰습니다.

## 3. 해결 방향

핵심은 Optional 플래그만으로 판단하지 않고, AST declaration 정보를 함께 확인하는 것이었습니다.
그리하여 `paramSymbol.valueDeclaration`이 파라미터 노드인지 확인하고, `dotDotDotToken`이 존재하면 rest parameter로 판단해 해당 경로에서는 `useless default` 리포트를 중단하도록 했습니다.
이 방식은 기존 판단 흐름을 유지하면서, 오탐이 발생하는 지점만 정확히 보정할 수 있다고 판단했습니다.

## 4. 코드 수정

수정 파일: `packages/eslint-plugin/src/rules/no-useless-default-assignment.ts`

```ts
if (paramIndex < params.length) {
  const paramSymbol = params[paramIndex];
  // 아래 Optional 체크 로직을 타기 전에 rest parameter를 먼저 판별해 false positive를 방지
  if (
    paramSymbol.valueDeclaration &&
    ts.isParameter(paramSymbol.valueDeclaration) &&
    paramSymbol.valueDeclaration.dotDotDotToken != null
  ) {
    return;
  }

  if ((paramSymbol.flags & ts.SymbolFlags.Optional) === 0) {
    const paramType = checker.getTypeOfSymbol(paramSymbol);
    if (!canBeUndefined(paramType)) {
      reportUselessDefaultAssignment(node, 'parameter');
    }
  }
}
```

## 5. 테스트 보강

수정 파일: `packages/eslint-plugin/tests/rules/no-useless-default-assignment.test.ts`

- 이슈 재현 코드를 `valid` 케이스로 추가했습니다.
- 후속 커밋에서 이슈 링크 주석(`// https://github.com/typescript-eslint/typescript-eslint/issues/11911`)을 넣어 테스트 의도를 명확히 했습니다.

```ts
// https://github.com/typescript-eslint/typescript-eslint/issues/11911
`
  const run = (cb: (...args: unknown[]) => void) => cb();
  const cb = (p: boolean = true) => null;
  run(cb);
  run((p: boolean = true) => null);
`,
```

## 6. 결과 및 기여

- rest parameter 대응 상황에서 발생하던 false positive를 제거했습니다.
- `no-useless-default-assignment` 규칙의 정확도를 개선했습니다.
- 동일 패턴의 회귀를 방지할 수 있도록 테스트를 보강했습니다.

## 7. 학습 포인트

- `no-useless-default-assignment`가 불필요한 초기값 할당을 경고하는 규칙이라는 점을 명확히 이해했습니다.
- false positive는 정상 코드를 오류로 판단하는 경우라는 점을 실제 이슈 맥락에서 체감했습니다.
- AST는 TypeScript 코드를 트리 구조로 표현한 객체이며, 규칙 구현에서 이 AST 정보가 핵심이라는 점을 확인했습니다.
- `SymbolFlags`만으로는 한계가 있어 declaration(`valueDeclaration`, `dotDotDotToken`)을 함께 확인해야 정확한 판정이 가능하다는 점을 학습했습니다.
