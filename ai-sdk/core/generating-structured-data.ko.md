# 구조화된 데이터 생성하기

텍스트 생성이 유용할 때도 있지만, 많은 실제 사용 사례에서는 구조화된 데이터를 생성해야 할 때가 많아.
예를 들어 텍스트에서 정보를 추출하거나, 데이터를 분류하거나, 합성 데이터를 생성하는 작업 같은 것들.

많은 언어 모델은 "JSON 모드"나 "툴"을 통해 구조화된 데이터를 생성할 수 있어.
하지만 LLM이 잘못되거나 불완전한 구조를 내보낼 수 있기 때문에, 스키마를 직접 정의하고 결과를 검증해야 해.

AI SDK는 다양한 모델 제공자 전반에서 구조화된 객체 생성을 표준화하기 위해
[`generateObject`](/docs/reference/ai-sdk-core/generate-object)와
[`streamObject`](/docs/reference/ai-sdk-core/stream-object) 함수를 제공해.
이 두 함수는 `array`, `object`, `enum`, `no-schema` 같은 다양한 출력 전략과,
`auto`, `tool`, `json` 같은 다양한 생성 모드를 지원해.
또한 원하는 데이터 형태를 지정하기 위해 [Zod 스키마](/docs/reference/ai-sdk-core/zod-schema),
[Valibot](/docs/reference/ai-sdk-core/valibot-schema), [JSON 스키마](/docs/reference/ai-sdk-core/json-schema)를 사용할 수 있고,
모델은 그 구조에 맞는 데이터를 생성하게 돼.

<Note>
  Zod 객체를 AI SDK 함수에 직접 전달하거나 `zodSchema` 헬퍼를 사용할 수 있어.
</Note>

## Generate Object

`generateObject`는 프롬프트로부터 구조화된 데이터를 생성해.
스키마는 생성된 데이터를 검증하는 데도 사용되므로 타입 안정성과 정확성을 보장해.

```ts
import { generateObject } from 'ai';
import { z } from 'zod';

const { object } = await generateObject({
  model: 'openai/gpt-4.1',
  schema: z.object({
    recipe: z.object({
      name: z.string(),
      ingredients: z.array(z.object({ name: z.string(), amount: z.string() })),
      steps: z.array(z.string()),
    }),
  }),
  prompt: 'Generate a lasagna recipe.',
});
```

<Note>
  `generateObject` 사용 예시는 [여기](#more-examples)에서 확인해봐.
</Note>

### 응답 헤더와 바디 접근하기

가끔은 모델 제공자의 전체 응답(특정 헤더나 바디 내용 등)에 접근해야 할 때가 있어.

`response` 프로퍼티를 통해 원본 응답 헤더와 바디에 접근할 수 있어:

```ts
import { generateObject } from 'ai';

const result = await generateObject({
  // ...
});

console.log(JSON.stringify(result.response.headers, null, 2));
console.log(JSON.stringify(result.response.body, null, 2));
```

## Stream Object

구조화된 데이터를 반환하는 건 복잡도가 높아서, 인터랙티브한 사용 사례에서는 응답 속도가 느릴 수 있어.
[`streamObject`](/docs/reference/ai-sdk-core/stream-object) 함수를 사용하면 모델이 생성하는 즉시 스트리밍 받을 수 있어.

```ts
import { streamObject } from 'ai';

const { partialObjectStream } = streamObject({
  // ...
});

// async iterable로 사용하기
for await (const partialObject of partialObjectStream) {
  console.log(partialObject);
}
```

`streamObject`는 React Server Components(참고: [Generative UI](../ai-sdk-rsc)))나 [`useObject`](/docs/reference/ai-sdk-ui/use-object) 훅과 조합해 생성된 UI를 스트리밍할 때도 사용할 수 있어.

<Note>`streamObject` 사용 예시는 [여기](#more-examples)에서 확인해봐.</Note>

### `onError` 콜백

`streamObject`는 즉시 스트리밍을 시작해.
서버 크래시를 막기 위해 에러는 throw되지 않고 스트림의 일부가 돼.

에러를 로깅하려면 에러 발생 시 호출되는 `onError` 콜백을 제공하면 돼.

```tsx highlight="5-7"
import { streamObject } from 'ai';

const result = streamObject({
  // ...
  onError({ error }) {
    console.error(error); // 여기서 에러 로깅
  },
});
```

## 출력 전략(Output Strategy)

두 함수는 `array`, `object`, `enum`, `no-schema` 등 다양한 출력 전략을 지원해.

### Object

기본 출력 전략은 `object`야. 생성된 데이터를 객체로 반환해.
기본값을 쓰려면 별도 설정이 필요 없어.

### Array

여러 개의 객체를 생성하려면 출력 전략을 `array`로 설정해.
`array` 전략을 사용할 때 스키마는 배열 요소의 형태를 정의해.
또한 `streamObject`에서는 `elementStream`으로 생성된 배열 요소를 스트리밍할 수 있어.

```ts highlight="7,18"
import { openai } from '@ai-sdk/openai';
import { streamObject } from 'ai';
import { z } from 'zod';

const { elementStream } = streamObject({
  model: openai('gpt-4.1'),
  output: 'array',
  schema: z.object({
    name: z.string(),
    class: z.string().describe('Character class, e.g. warrior, mage, or thief.'),
    description: z.string(),
  }),
  prompt: 'Generate 3 hero descriptions for a fantasy role playing game.',
});

for await (const hero of elementStream) {
  console.log(hero);
}
```

### Enum

분류 같은 작업에서 특정 enum 값을 생성하고 싶다면 출력 전략을 `enum`으로 설정하고,
`enum` 파라미터로 가능한 값 목록을 제공하면 돼.

<Note>`enum` 출력은 `generateObject`에서만 지원돼.</Note>

```ts highlight="5-6"
import { generateObject } from 'ai';

const { object } = await generateObject({
  model: 'openai/gpt-4.1',
  output: 'enum',
  enum: ['action', 'comedy', 'drama', 'horror', 'sci-fi'],
  prompt:
    'Classify the genre of this movie plot: ' +
    '"A group of astronauts travel through a wormhole in search of a ' +
    'new habitable planet for humanity."',
});
```

### No Schema

어떤 경우에는 스키마를 사용하고 싶지 않을 수도 있어(예: 매우 동적인 사용자 요청).
그럴 때는 `output: 'no-schema'`로 설정하고 `schema` 파라미터를 생략하면 돼.

```ts highlight="6"
import { openai } from '@ai-sdk/openai';
import { generateObject } from 'ai';

const { object } = await generateObject({
  model: openai('gpt-4.1'),
  output: 'no-schema',
  prompt: 'Generate a lasagna recipe.',
});
```

## 스키마 이름과 설명

스키마의 이름과 설명을 선택적으로 지정할 수 있어. 일부 제공자는 이를 도구/스키마 이름을 통한 추가 LLM 가이드로 활용해.

```ts highlight="6-7"
import { generateObject } from 'ai';
import { z } from 'zod';

const { object } = await generateObject({
  model: 'openai/gpt-4.1',
  schemaName: 'Recipe',
  schemaDescription: 'A recipe for a dish.',
  schema: z.object({
    name: z.string(),
    ingredients: z.array(z.object({ name: z.string(), amount: z.string() })),
    steps: z.array(z.string()),
  }),
  prompt: 'Generate a lasagna recipe.',
});
```

## Reasoning 접근

생성된 객체를 만들기 위해 모델이 사용한 추론 내용을 `reasoning` 프로퍼티를 통해 확인할 수 있어(제공되는 경우).

```ts
import { openai, OpenAIResponsesProviderOptions } from '@ai-sdk/openai';
import { generateObject } from 'ai';
import { z } from 'zod';

const result = await generateObject({
  model: openai('gpt-5'),
  schema: z.object({
    recipe: z.object({
      name: z.string(),
      ingredients: z.array(
        z.object({
          name: z.string(),
          amount: z.string(),
        }),
      ),
      steps: z.array(z.string()),
    }),
  }),
  prompt: 'Generate a lasagna recipe.',
  providerOptions: {
    openai: {
      strictJsonSchema: true,
      reasoningSummary: 'detailed',
    } satisfies OpenAIResponsesProviderOptions,
  },
});

console.log(result.reasoning);
```

## 에러 처리

`generateObject`가 유효한 객체를 생성하지 못하면 [`AI_NoObjectGeneratedError`](/docs/reference/ai-sdk-errors/ai-no-object-generated-error)를 던져.

이 에러는 모델이 스키마에 부합하는 파싱 가능한 객체를 생성하지 못했을 때 발생해. 가능한 원인:

- 모델이 응답 생성에 실패함
- 생성된 응답을 파싱할 수 없음
- 생성된 응답이 스키마 검증을 통과하지 못함

로그에 도움이 되도록 에러는 다음 정보를 포함해:

- `text`: 모델이 생성한 텍스트(원본 텍스트 또는 툴 호출 텍스트)
- `response`: 응답 ID, 타임스탬프, 모델 등 메타데이터
- `usage`: 토큰 사용량
- `cause`: 에러 원인(예: JSON 파싱 에러) — 세부 에러 처리에 활용 가능

```ts
import { generateObject, NoObjectGeneratedError } from 'ai';

try {
  await generateObject({ model, schema, prompt });
} catch (error) {
  if (NoObjectGeneratedError.isInstance(error)) {
    console.log('NoObjectGeneratedError');
    console.log('Cause:', error.cause);
    console.log('Text:', error.text);
    console.log('Response:', error.response);
    console.log('Usage:', error.usage);
  }
}
```

## 잘못된/깨진 JSON 복구하기

<Note type="warning">
  `repairText`는 실험적 기능이야. 향후 변경될 수 있어.
</Note>

가끔 모델이 잘못되거나 깨진 JSON을 생성할 수 있어.
이럴 때 `repairText` 함수를 사용해 JSON 복구를 시도할 수 있어.

이 함수는 `JSONParseError` 또는 `TypeValidationError`와, 모델이 생성한 텍스트를 입력으로 받아.
그 후 텍스트를 복구해 반환하도록 시도할 수 있어.

```ts highlight="7-10"
import { generateObject } from 'ai';

const { object } = await generateObject({
  model,
  schema,
  prompt,
  experimental_repairText: async ({ text, error }) => {
    // 예: 닫는 중괄호를 추가
    return text + '}';
  },
});
```

## `generateText`/`streamText`로 구조화된 출력 생성하기

`generateText`와 `streamText`에서도 `experimental_output` 설정을 사용해 구조화된 데이터를 생성할 수 있어.

<Note>
  일부 모델(예: OpenAI)은 구조화 출력과 툴 호출을 동시에 지원해.
  이는 `generateText`/`streamText`에서만 가능해.
</Note>

<Note type="warning">
  `generateText`/`streamText`의 구조화 출력은 실험적 기능이며 향후 변경될 수 있어.
</Note>

### `generateText`

```ts highlight="2,4-18"
// experimental_output은 스키마에 맞는 구조화 객체야:
const { experimental_output } = await generateText({
  // ...
  experimental_output: Output.object({
    schema: z.object({
      name: z.string(),
      age: z.number().nullable().describe('Age of the person.'),
      contact: z.object({
        type: z.literal('email'),
        value: z.string(),
      }),
      occupation: z.object({
        type: z.literal('employed'),
        company: z.string(),
        position: z.string(),
      }),
    }),
  }),
  prompt: 'Generate an example person for testing.',
});
```

### `streamText`

```ts highlight="2,4-18"
// experimental_partialOutputStream은 부분 객체를 스트리밍해:
const { experimental_partialOutputStream } = await streamText({
  // ...
  experimental_output: Output.object({
    schema: z.object({
      name: z.string(),
      age: z.number().nullable().describe('Age of the person.'),
      contact: z.object({
        type: z.literal('email'),
        value: z.string(),
      }),
      occupation: z.object({
        type: z.literal('employed'),
        company: z.string(),
        position: z.string(),
      }),
    }),
  }),
  prompt: 'Generate an example person for testing.',
});
```
