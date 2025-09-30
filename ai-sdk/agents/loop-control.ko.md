# 루프 제어(Loop Control)

AI SDK로 에이전트를 구축할 때, 에이전트 루프의 각 단계에서 **실행 흐름**과 **설정**을 모두 제어할 수 있어. AI SDK는 두 가지 매개변수로 내장된 루프 제어를 제공해: 단계 종료 조건을 정의하는 `stopWhen`, 단계 사이에서 모델/도구/메시지 등을 수정하는 `prepareStep`.

이 두 매개변수는 다음 모두에서 사용할 수 있어:

- AI SDK Core의 `generateText`와 `streamText`
- 객체지향 에이전트 구현을 위한 `Agent` 클래스

## 정지 조건(Stop Conditions)

`stopWhen`은 마지막 단계에서 도구 결과가 있을 때 **생성(Generation)**을 언제 멈출지 제어해. 기본값은 한 단계만 허용하는 `stepCountIs(1)`야.

`stopWhen`을 지정하면, 도구 호출 이후에도 정지 조건을 만족할 때까지 SDK가 계속해서 응답 생성을 이어가. 배열로 여러 조건을 주면, 그중 하나라도 만족하면 루프를 멈춰.

### 내장 조건 사용

```ts
import { generateText, stepCountIs } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    // your tools
  },
  stopWhen: stepCountIs(10), // 최대 10단계 수행 후 중지
  prompt: 'Analyze this dataset and create a summary report',
});
```

### 여러 조건 결합

여러 정지 조건을 결합해, 조건 중 **하나라도** 충족하면 루프를 중지시킬 수 있어:

```ts
import { generateText, stepCountIs, hasToolCall } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    // your tools
  },
  stopWhen: [
    stepCountIs(10), // 최대 10단계
    hasToolCall('someTool'), // 'someTool' 호출 후 중지
  ],
  prompt: 'Research and analyze the topic',
});
```

### 커스텀 조건 만들기

특정 요구사항에 맞춘 커스텀 정지 조건을 만들 수 있어:

```ts
import { generateText, StopCondition, ToolSet } from 'ai';

const tools = {
  // your tools
} satisfies ToolSet;

const hasAnswer: StopCondition<typeof tools> = ({ steps }) => {
  // 텍스트에 "ANSWER:"가 포함되면 중지
  return steps.some((step) => step.text?.includes('ANSWER:')) ?? false;
};

const result = await generateText({
  model: 'openai/gpt-4o',
  tools,
  stopWhen: hasAnswer,
  prompt: 'Find the answer and respond with "ANSWER: [your answer]"',
});
```

커스텀 조건은 모든 단계의 정보를 받을 수 있어:

```ts
const budgetExceeded: StopCondition<typeof tools> = ({ steps }) => {
  const totalUsage = steps.reduce(
    (acc, step) => ({
      inputTokens: acc.inputTokens + (step.usage?.inputTokens ?? 0),
      outputTokens: acc.outputTokens + (step.usage?.outputTokens ?? 0),
    }),
    { inputTokens: 0, outputTokens: 0 },
  );

  const costEstimate = (totalUsage.inputTokens * 0.01 + totalUsage.outputTokens * 0.03) / 1000;
  return costEstimate > 0.5; // 비용이 $0.50을 넘으면 중지
};
```

## 준비 단계(Prepare Step)

`prepareStep` 콜백은 루프의 **각 단계 시작 전**에 실행돼. 아무 것도 반환하지 않으면 초기 설정을 그대로 사용해. 이 훅을 이용해 설정을 바꾸거나, 컨텍스트를 관리하거나, 실행 기록에 따른 동적 동작을 구현할 수 있어.

### 동적 모델 선택

단계 요구사항에 따라 모델을 바꿔:

```ts
import { generateText } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4o-mini', // 기본 모델
  tools: {
    // your tools
  },
  prepareStep: async ({ stepNumber, messages }) => {
    // 일정 단계 이후 복잡한 추론에는 더 강한 모델 사용
    if (stepNumber > 2 && messages.length > 10) {
      return {
        model: 'openai/gpt-4o',
      };
    }
    // 기본 설정 유지
    return {};
  },
});
```

### 컨텍스트 관리

긴 루프에서 대화 이력을 관리해 컨텍스트 초과를 방지:

```ts
const result = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    // your tools
  },
  prepareStep: async ({ messages }) => {
    // 최근 메시지만 유지
    if (messages.length > 20) {
      return {
        messages: [
          messages[0], // 시스템 메시지는 유지
          ...messages.slice(-10), // 마지막 10개만 유지
        ],
      };
    }
    return {};
  },
});
```

### 도구 선택

단계별로 사용 가능한 도구를 통제해:

```ts
const result = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    search: searchTool,
    analyze: analyzeTool,
    summarize: summarizeTool,
  },
  prepareStep: async ({ stepNumber, steps }) => {
    // 검색 단계 (0~2단계)
    if (stepNumber <= 2) {
      return {
        activeTools: ['search'],
        toolChoice: 'required',
      };
    }

    // 분석 단계 (3~5단계)
    if (stepNumber <= 5) {
      return {
        activeTools: ['analyze'],
      };
    }

    // 요약 단계 (6단계 이상)
    return {
      activeTools: ['summarize'],
      toolChoice: 'required',
    };
  },
});
```

특정 도구 사용을 강제할 수도 있어:

```ts
prepareStep: async ({ stepNumber }) => {
  if (stepNumber === 0) {
    // 1단계에서 검색 도구 강제
    return {
      toolChoice: { type: 'tool', toolName: 'search' },
    };
  }

  if (stepNumber === 5) {
    // 5단계에서 요약 도구 강제
    return {
      toolChoice: { type: 'tool', toolName: 'summarize' },
    };
  }

  return {};
};
```

### 메시지 수정

모델로 보내기 전에 메시지를 가공해:

```ts
const result = await generateText({
  model: 'openai/gpt-4o',
  messages,
  tools: {
    // your tools
  },
  prepareStep: async ({ messages, stepNumber }) => {
    // 도구 결과를 요약해 토큰 사용량 절감
    const processedMessages = messages.map((msg) => {
      if (msg.role === 'tool' && msg.content.length > 1000) {
        return {
          ...msg,
          content: summarizeToolResult(msg.content),
        };
      }
      return msg;
    });

    return { messages: processedMessages };
  },
});
```

## 단계 정보 접근

`stopWhen`과 `prepareStep`은 현재 실행에 대한 상세 정보를 전달받아:

```ts
prepareStep: async ({
  model, // 현재 모델 설정
  stepNumber, // 현재 단계 번호(0부터 시작)
  steps, // 지금까지의 모든 단계와 결과
  messages, // 이번 단계에서 모델로 보낼 메시지
}) => {
  // 이전 도구 호출/결과에 접근
  const previousToolCalls = steps.flatMap((step) => step.toolCalls);
  const previousResults = steps.flatMap((step) => step.toolResults);

  // 실행 이력 기반 의사결정
  if (previousToolCalls.some((call) => call.toolName === 'dataAnalysis')) {
    return {
      toolChoice: { type: 'tool', toolName: 'reportGenerator' },
    };
  }

  return {};
},
```

## 수동 루프 제어

에이전트 루프를 완전히 직접 제어해야 하는 경우, `stopWhen`/`prepareStep` 대신 **직접 루프를 구현**해. 복잡한 워크플로에서 최대한의 유연성을 제공해.

[쿠크북의 수동 에이전트 루프 예시](/cookbook/node/manual-agent-loop)도 참고해.
