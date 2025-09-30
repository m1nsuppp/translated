# 에이전트(Agents)

에이전트는 **LLM(대규모 언어 모델)**이 **도구(tools)**를 **루프(loop)** 안에서 사용해 과업을 수행하는 시스템이야.

각 구성 요소의 역할은 아래와 같아:

- **LLM**: 입력(텍스트)을 처리하고 다음에 취할 행동을 결정
- **도구**: 텍스트 생성만으로는 할 수 없는 작업을 확장(예: 파일 읽기, API 호출, DB 쓰기)
- **루프**: 실행을 오케스트레이션
  - **컨텍스트 관리**: 대화 히스토리를 유지하고 각 단계에서 모델이 볼 입력을 결정
  - **정지 조건**: 루프(작업)를 언제 종료할지 결정

## 빌딩 블록

이 기본 컴포넌트들을 조합해서 점점 더 정교한 시스템을 만들 수 있어.

### 단일 단계 생성(Single-Step Generation)

LLM을 한 번 호출해 응답을 받는 방식. 분류나 간단한 텍스트 생성처럼 직선적인 작업에 사용해.

```ts
import { generateText } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4o',
  prompt: 'Classify this sentiment: "I love this product!"',
});
```

### 도구 사용(Tool Usage)

도구를 통해 LLM의 능력을 외부 시스템으로 확장해. 도구는 컨텍스트 보강(파일/DB 조회 등) 또는 액션 수행(이메일 전송/레코드 업데이트 등)에 쓰일 수 있어.

```ts
import { generateText, tool } from 'ai';
import { z } from 'zod';

const result = await generateText({
  model: 'openai/gpt-4o',
  prompt: 'What is the weather in San Francisco?',
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      parameters: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
  },
});

console.log(result.toolResults);
```

### 다단계 도구 사용(Agents)

복잡한 문제에서는 LLM이 여러 단계를 거치며 여러 번 도구를 호출할 수 있어. 작업에 따라 모델이 도구 호출의 순서/횟수를 스스로 결정해.

```ts
import { generateText, stepCountIs, tool } from 'ai';
import { z } from 'zod';

const result = await generateText({
  model: 'openai/gpt-4o',
  prompt: 'What is the weather in San Francisco in celsius?',
  tools: {
    weather: tool({
      description: 'Get the weather in a location (in Fahrenheit)',
      parameters: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
    convertFahrenheitToCelsius: tool({
      description: 'Convert temperature from Fahrenheit to Celsius',
      parameters: z.object({
        temperature: z.number().describe('Temperature in Fahrenheit'),
      }),
      execute: async ({ temperature }) => {
        const celsius = Math.round((temperature - 32) * (5 / 9));
        return { celsius };
      },
    }),
  },
  stopWhen: stepCountIs(10), // 최대 10단계까지만 수행
});

console.log(result.text); // 출력 예: The weather in San Francisco is currently _°C.
```

모델은 다음처럼 동작할 수 있어:

1. `weather` 도구로 화씨 온도 조회
2. `convertFahrenheitToCelsius` 도구로 섭씨 변환
3. 변환된 온도로 텍스트 응답 생성

이 동작은 유연하며, 모델이 과업을 어떻게 이해했는지에 따라 접근 방식이 달라질 수 있어.

## 구현 방식

AI SDK는 에이전트를 만드는 두 가지 접근을 제공해.

### Agent Class

루프를 대신 처리해주는 객체 지향 추상화. 다음이 필요할 때 좋아:

- 에이전트 설정 재사용
- 보일러플레이트 최소화
- 일관된 에이전트 행동 구축

```ts
import { Experimental_Agent as Agent } from 'ai';

const myAgent = new Agent({
  model: 'openai/gpt-4o',
  tools: {
    // your tools here
  },
  stopWhen: stepCountIs(10), // 최대 10단계까지 수행
});

const result = await myAgent.generate({
  prompt: 'Analyze the latest sales data and create a summary report',
});

console.log(result.text);
```

[Agent 클래스 더 알아보기](/docs/agents/agent-class).

### 코어 함수(Core Functions)

`generateText` 또는 `streamText`를 도구와 함께 사용. 선택지는 다음과 같아:

**내장 루프** - SDK가 실행 사이클을 관리하도록 맡기기:

```ts
import { generateText, stepCountIs } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4o',
  prompt: 'Research machine learning trends and provide key insights',
  tools: {
    // your tools here
  },
  stopWhen: stepCountIs(10),
  prepareStep: ({ stepNumber }) => {
    // 각 단계 사이에 설정 수정
  },
  onStepFinish: (step) => {
    // 진행 모니터링/저장
  },
});
```

[루프 제어 더 알아보기](/docs/agents/loop-control).

**수동 루프** - 실행을 완전히 직접 제어하기:

```ts
import { generateText, ModelMessage } from 'ai';

const messages: ModelMessage[] = [{ role: 'user', content: '...' }];

let step = 0;
const maxSteps = 10;

while (step < maxSteps) {
  const result = await generateText({
    model: 'openai/gpt-4o',
    messages,
    tools: {
      // your tools here
    },
  });

  messages.push(...result.response.messages);

  if (result.text) {
    break; // 모델이 텍스트를 생성하면 종료
  }

  step++;
}
```

## 더 많은 제어가 필요할 때

에이전트는 강력하지만 비결정적이야. 신뢰할 수 있고 반복 가능한 결과가 필요하다면, 도구 호출과 일반 프로그래밍 패턴을 결합해:

- 명시적 분기를 위한 조건문
- 재사용 가능한 로직을 위한 함수
- 견고함을 위한 에러 처리
- 예측 가능성을 위한 명시적 제어 흐름

이 접근은 AI의 장점을 살리면서도 중요한 경로에 대한 통제를 유지하게 해.

[워크플로 패턴 살펴보기](/docs/agents/workflows).
