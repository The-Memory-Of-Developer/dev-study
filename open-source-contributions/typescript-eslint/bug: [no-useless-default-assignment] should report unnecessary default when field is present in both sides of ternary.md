# bug: [no-useless-default-assignment] should report unnecessary default when field is present in both sides of ternary

## 링크

- Issue: https://github.com/typescript-eslint/typescript-eslint/issues/11980
- PR: https://github.com/typescript-eslint/typescript-eslint/pull/11984
- Changes: https://github.com/typescript-eslint/typescript-eslint/pull/11984/changes

## 1. 문제 현상(재현 코드)

```ts
const { a = 'baz' } = Math.random() < 0.5 ? { a: 'foo' } : { a: 'bar' };
```

위 코드에서는 어떤 분기가 선택되더라도 a가 항상 존재하므로 a = 'baz'의 기본값은 실제로 사용되지 않습니다.
하지만 기존 @typescript-eslint/no-useless-default-assignment 규칙에서는 이 케이스에서 경고가 발생하지 않았습니다.

## 2. 원인 분석

원인을 확인하기 위해 `packages/eslint-plugin/src/rules/no-useless-default-assignment.ts`의 프로퍼티 존재 판정 흐름을 점검했습니다.

- 기존 로직은 단일 객체 형태 또는 제한된 패턴 중심으로 필드 존재 여부를 판단했습니다.
- 조건식(ternary)처럼 분기가 있는 표현식에서는, 각 분기에 동일 필드가 있는지를 모두 확인하는 로직이 부족했습니다.
- 결과적으로 양쪽 분기에 필드가 있어도 "항상 존재"를 확정하지 못해, 불필요한 기본값을 놓쳤습니다.

## 3. 해결 방향

핵심은 "조건식의 모든 분기에서 대상 필드가 존재하는지"를 재귀적으로 확인하는 것이었습니다.
이를 위해 분기형 표현식을 만났을 때 단일 경로만 보지 않고 각 branch를 순회하며 동일 필드 존재 여부를 검사한 뒤, 모든 분기에서 true일 때만 "기본값이 불필요"하다고 판단하도록 보강했습니다.
이러한 로직을 통해 ternary가 중첩되어도 같은 기준으로 판정할 수 있게 되었습니다.

## 4. 코드 수정

수정 파일: `packages/eslint-plugin/src/rules/no-useless-default-assignment.ts`

핵심 수정은 "optional 프로퍼티 예외 처리 강화 + 모든 분기 재귀 검사 + 키 판별 확장" 세 가지였습니다.

```ts
if (tsutils.isSymbolFlagSet(symbol, ts.SymbolFlags.Optional)) {
  const parent = objectPattern.parent;
  if (
    // 구조 분해 대상이 변수 선언 초기화이고,
    parent.type === AST_NODE_TYPES.VariableDeclarator &&
    parent.init &&
    // 초기화 표현식이 조건식(삼항식) 경로를 포함하는지 확인
    hasConditionalInitializer(objectPattern)
  ) {
    const propertyName = getPropertyName(node.key);

    // 키 이름을 확정할 수 없거나, 모든 분기에 대상 키가 존재하지 않으면 기존처럼 리포트하지 않음
    if (!propertyName || !hasPropertyInAllBranches(parent.init, propertyName)) {
      return null;
    }
  }
}
```

```ts
function hasPropertyInAllBranches(expr: TSESTree.Expression, propertyName: string): boolean {
  return (
    // 현재 표현식이 객체 리터럴이면 해당 키 존재 여부 검사
    (expression.type === AST_NODE_TYPES.ObjectExpression &&
      expression.properties.some(
        (prop) =>
          prop.type === AST_NODE_TYPES.Property && getPropertyName(prop.key) === propertyName,
      )) ||
    // 현재 표현식이 조건식이면 consequent/alternate 모두 재귀 검사
    (expression.type === AST_NODE_TYPES.ConditionalExpression &&
      hasPropertyInAllBranches(expression.consequent, propertyName) &&
      hasPropertyInAllBranches(expression.alternate, propertyName))
  );
}
```

```ts
function getPropertyName(key: TSESTree.PropertyName): string | null {
  switch (key.type) {
    case AST_NODE_TYPES.Identifier:
      return key.name;
    case AST_NODE_TYPES.Literal:
      return String(key.value);
    case AST_NODE_TYPES.TemplateLiteral:
      // 정적 템플릿 리터럴(`a`)도 키로 인식
      return key.expressions.length ? null : key.quasis[0].value.cooked;
    default:
      return null;
  }
}
```

위 변경으로 삼항식/중첩 삼항식에서 "모든 분기에 공통으로 존재하는 필드"를 안정적으로 판별해, 불필요한 기본값 할당을 누락 없이 탐지하도록 개선했습니다.

## 5. 테스트 보강

수정 파일: `packages/eslint-plugin/tests/rules/no-useless-default-assignment.test.ts`

- 작성한 테스트는 크게 `invalid(반드시 리포트되어야 하는 경우)`와 `valid(리포트하면 안 되는 경우)`를 모두 포함했습니다.
- 또한 구현 중간에 추가했던 탐색/경계 케이스도 함께 검토해, false positive와 false negative를 동시에 점검했습니다.

추가한 `invalid` 테스트(리포트되어야 하는 케이스):

```ts
const { a = 'baz' } = Math.random() < 0.5 ? { a: 'foo' } : { a: 'bar' };
const { a = 'baz' } =
  Math.random() < 0.5 ? { a: 'foo' } : Math.random() > 0.2 ? { a: 'bar' } : { a: 'qux' };
const { a = 'baz' } = cond ? { ['a']: 'foo' } : { ['a']: 'bar' };
const { a = 'baz' } = cond ? { a() {} } : { a: 'bar' };
const { a = 'b' } = Math.random() < 0.5 ? { [`a`]: 'a' } : { a: 'b' };
```

추가한 `valid` 테스트(리포트되면 안 되는 케이스):

```ts
const { a = 'baz' } = cond ? {} : { a: 'bar' };
const { a = 'baz' } = cond ? foo : { a: 'bar' };
const { a = 'baz' } = foo && { a: 'bar' };
const { a = 'baz' } = cond ? { a: 'foo', ...extra } : { a: 'bar' };
const { a = 'baz' } = cond ? { ...foo } : { a: 'bar' };
const key = Math.random() > 0.5 ? 'a' : 'b';
const { a = 'baz' } = cond ? { [key]: 'foo' } : { [key]: 'bar' };
const { a = 'baz' } = cond ? foo && { a: 'bar' } : { a: 'baz' };
const obj: unknown = { a: 'bar' };
const { a = 'baz' } = cond ? obj : { a: 'bar' };
const sym = Symbol('a');
const { a = 'baz' } = cond ? { [sym]: 'foo' } : { [sym]: 'bar' };
const { a = 'baz' } = cond ? { [`a${1}`]: 'foo' } : { a: 'bar' };
```

작성 후 정리/조정한 탐색 케이스:

```ts
const key = 'a';
const { a = 'baz' } = cond ? { [key]: 'foo' } : { [key]: 'bar' };
const foo = Math.random() > 0.5 ? 1 : 2;
const { a = 'baz' } = foo;
const obj: unknown = { ...foo };
const { a = 'baz' } = cond ? obj : { a: 'bar' };
const obj2: unknown = [];
const { a = 'baz' } = cond ? obj2 : { a: 'bar' };
const key2: unknown = Symbol('a');
const obj3: unknown = { [key2 as any]: 42 };
const { a = 'baz' } = cond ? obj3 : { a: 'bar' };
```

위 테스트 세트를 통해 조건식 분기, 중첩 조건식, computed key, spread, unknown 타입, 메서드 프로퍼티 등 다양한 경계조건에서 규칙의 동작을 확인했습니다.

## 6. 결과 및 기여

- 삼항 연산자 분기에서 발생하던 false negative를 제거했습니다.
- `no-useless-default-assignment` 규칙의 탐지 일관성을 개선했습니다.
- 분기형 객체 표현식에 대한 회귀 테스트를 보강해 안정성을 높였습니다.

## 7. 학습 포인트

- `no-useless-default-assignment`는 "기본값이 실제로 필요한 경로가 있는지"를 정밀하게 판단해야 하는 규칙임을 확인했습니다.
- false positive뿐 아니라 false negative도 규칙 신뢰도를 크게 저하시킨다는 점을 체감했습니다.
- 조건식/중첩 조건식 분석에서는 단일 노드 체크보다 "모든 분기"를 확인하는 재귀 접근이 효과적이라는 점을 학습했습니다.
- AST 기반 규칙 구현에서 분기 구조를 안전하게 순회하는 설계가 핵심임을 확인했습니다.
