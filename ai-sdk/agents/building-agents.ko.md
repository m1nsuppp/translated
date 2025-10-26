# 에이전트 구축

Agent 클래스는 LLM 구성, 도구 및 동작을 재사용 가능한 구성 요소로 캡슐화하는 구조화된 방법을 제공합니다. 에이전트 루프를 자동으로 처리하여 LLM이 복잡한 작업을 수행하기 위해 도구를 순차적으로 여러 번 호출할 수 있도록 합니다. 에이전트를 한 번 정의하고 애플리케이션 전체에서 사용하세요.

## Agent 클래스를 사용하는 이유는?

AI 애플리케이션을 구축할 때 다음이 필요한 경우가 많습니다:

- **구성 재사용** - 애플리케이션의 여러 부분에서 동일한 모델 설정, 도구 및 프롬프트 사용
- **일관성 유지** - 코드베이스 전체에서 동일한 동작과 기능 보장
- **API 라우트 단순화** - 엔드포인트의 보일러플레이트 감소
- **타입 안전성** - 에이전트의 도구와 출력에 대한 완전한 TypeScript 지원

Agent 클래스는 에이전트의 동작을 정의할 단일 위치를 제공합니다.

## 에이전트 생성

원하는 구성으로 Agent 클래스를 인스턴스화하여 에이전트를 정의합니다:

```ts
import { Experimental_Agent as Agent } from 'ai';

const myAgent = new Agent({
  model: 'openai/gpt-4o',
  system: 'You are a helpful assistant.',
  tools: {
    // 여기에 도구 추가
  },
});
```

## 구성 옵션

Agent 클래스는 `generateText` 및 `streamText`와 동일한 모든 설정을 받습니다. 다음을 구성할 수 있습니다:

### 모델 및 시스템 프롬프트

```ts
import { Experimental_Agent as Agent } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  system: 'You are an expert software engineer.',
});
```

### 도구

에이전트가 작업을 수행하는 데 사용할 수 있는 도구를 제공합니다:

```ts
import { Experimental_Agent as Agent, tool } from 'ai';
import { z } from 'zod';

const codeAgent = new Agent({
  model: 'openai/gpt-4o',
  tools: {
    runCode: tool({
      description: 'Execute Python code',
      inputSchema: z.object({
        code: z.string(),
      }),
      execute: async ({ code }) => {
        // 코드를 실행하고 결과 반환
        return { output: 'Code executed successfully' };
      },
    }),
  },
});
```

### 루프 제어

기본적으로 에이전트는 단일 스텝으로 실행됩니다(`stopWhen: stepCountIs(1)`). 각 스텝에서 모델은 텍스트를 생성하거나 도구를 호출합니다. 텍스트를 생성하면 에이전트가 완료됩니다. 도구를 호출하면 AI SDK가 해당 도구를 실행합니다.

에이전트가 여러 도구를 순차적으로 호출할 수 있도록 하려면 더 많은 스텝을 허용하도록 `stopWhen`을 구성하세요. 각 도구 실행 후, 에이전트는 모델이 다른 도구를 호출하거나 텍스트를 생성할 수 있는 새로운 생성을 트리거합니다:

```ts
import { Experimental_Agent as Agent, stepCountIs } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  stopWhen: stepCountIs(20), // 최대 20 스텝 허용
});
```

각 스텝은 하나의 생성을 나타냅니다(텍스트 또는 도구 호출로 이어짐). 루프는 다음까지 계속됩니다:

- 모델이 도구를 호출하는 대신 텍스트를 생성하거나,
- 중지 조건이 충족될 때

여러 조건을 결합할 수 있습니다:

```ts
import { Experimental_Agent as Agent, stepCountIs } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  stopWhen: [
    stepCountIs(20), // 최대 20 스텝
    yourCustomCondition(), // 중지할 시점에 대한 사용자 정의 로직
  ],
});
```

[루프 제어 및 중지 조건](/docs/agents/loop-control)에 대해 자세히 알아보세요.

### 도구 선택

에이전트가 도구를 사용하는 방법을 제어합니다:

```ts
import { Experimental_Agent as Agent } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  tools: {
    // 여기에 도구 추가
  },
  toolChoice: 'required', // 도구 사용 강제
  // 또는 toolChoice: 'none'으로 도구 비활성화
  // 또는 toolChoice: 'auto'(기본값)로 모델이 결정하도록 함
});
```

특정 도구의 사용을 강제할 수도 있습니다:

```ts
import { Experimental_Agent as Agent } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  tools: {
    weather: weatherTool,
    cityAttractions: attractionsTool,
  },
  toolChoice: {
    type: 'tool',
    toolName: 'weather', // weather 도구 사용 강제
  },
});
```

### 구조화된 출력

구조화된 출력 스키마를 정의합니다:

```ts
import { Experimental_Agent as Agent, Output, stepCountIs } from 'ai';
import { z } from 'zod';

const analysisAgent = new Agent({
  model: 'openai/gpt-4o',
  experimental_output: Output.object({
    schema: z.object({
      sentiment: z.enum(['positive', 'neutral', 'negative']),
      summary: z.string(),
      keyPoints: z.array(z.string()),
    }),
  }),
  stopWhen: stepCountIs(10),
});

const { experimental_output: output } = await analysisAgent.generate({
  prompt: 'Analyze customer feedback from the last quarter',
});
```

## 시스템 프롬프트로 에이전트 동작 정의

시스템 프롬프트는 에이전트의 동작, 성격 및 제약 조건을 정의합니다. 모든 상호작용의 컨텍스트를 설정하고 에이전트가 사용자 쿼리에 응답하고 도구를 사용하는 방법을 안내합니다.

### 기본 시스템 프롬프트

에이전트의 역할과 전문성을 설정합니다:

```ts
const agent = new Agent({
  model: 'openai/gpt-4o',
  system:
    'You are an expert data analyst. You provide clear insights from complex data.',
});
```

### 상세한 동작 지침

에이전트 동작에 대한 구체적인 가이드라인을 제공합니다:

```ts
const codeReviewAgent = new Agent({
  model: 'openai/gpt-4o',
  system: `You are a senior software engineer conducting code reviews.

  Your approach:
  - Focus on security vulnerabilities first
  - Identify performance bottlenecks
  - Suggest improvements for readability and maintainability
  - Be constructive and educational in your feedback
  - Always explain why something is an issue and how to fix it`,
});
```

### 에이전트 동작 제약

경계를 설정하고 일관된 동작을 보장합니다:

```ts
const customerSupportAgent = new Agent({
  model: 'openai/gpt-4o',
  system: `You are a customer support specialist for an e-commerce platform.

  Rules:
  - Never make promises about refunds without checking the policy
  - Always be empathetic and professional
  - If you don't know something, say so and offer to escalate
  - Keep responses concise and actionable
  - Never share internal company information`,
  tools: {
    checkOrderStatus,
    lookupPolicy,
    createTicket,
  },
});
```

### 도구 사용 지침

에이전트가 사용 가능한 도구를 사용하는 방법을 안내합니다:

```ts
const researchAgent = new Agent({
  model: 'openai/gpt-4o',
  system: `You are a research assistant with access to search and document tools.

  When researching:
  1. Always start with a broad search to understand the topic
  2. Use document analysis for detailed information
  3. Cross-reference multiple sources before drawing conclusions
  4. Cite your sources when presenting information
  5. If information conflicts, present both viewpoints`,
  tools: {
    webSearch,
    analyzeDocument,
    extractQuotes,
  },
});
```

### 형식 및 스타일 지침

출력 형식과 커뮤니케이션 스타일을 제어합니다:

```ts
const technicalWriterAgent = new Agent({
  model: 'openai/gpt-4o',
  system: `You are a technical documentation writer.

  Writing style:
  - Use clear, simple language
  - Avoid jargon unless necessary
  - Structure information with headers and bullet points
  - Include code examples where relevant
  - Write in second person ("you" instead of "the user")

  Always format responses in Markdown.`,
});
```

## 에이전트 사용

정의된 에이전트는 세 가지 방법으로 사용할 수 있습니다:

### 텍스트 생성

일회성 텍스트 생성에는 `generate()`를 사용합니다:

```ts
const result = await myAgent.generate({
  prompt: 'What is the weather like?',
});

console.log(result.text);
```

### 텍스트 스트리밍

스트리밍 응답에는 `stream()`을 사용합니다:

```ts
const stream = myAgent.stream({
  prompt: 'Tell me a story',
});

for await (const chunk of stream.textStream) {
  console.log(chunk);
}
```

### UI 메시지에 응답

클라이언트 애플리케이션을 위한 API 응답을 생성하려면 `respond()`를 사용합니다:

```ts
// API 라우트에서 (예: app/api/chat/route.ts)
import { validateUIMessages } from 'ai';

export async function POST(request: Request) {
  const { messages } = await request.json();

  return myAgent.respond({
    messages: await validateUIMessages({ messages }),
  });
}
```

## 종단 간 타입 안전성

Agent의 `UIMessage` 타입을 추론할 수 있습니다:

```ts
import {
  Experimental_Agent as Agent,
  Experimental_InferAgentUIMessage as InferAgentUIMessage,
} from 'ai';

const myAgent = new Agent({
  // ... 구성
});

// UI 컴포넌트 또는 영속성을 위한 UIMessage 타입 추론
export type MyAgentUIMessage = InferAgentUIMessage<typeof myAgent>;
```

클라이언트 컴포넌트에서 `useChat`과 함께 이 타입을 사용합니다:

```tsx filename="components/chat.tsx"
'use client';

import { useChat } from '@ai-sdk/react';
import type { MyAgentUIMessage } from '@/agent/my-agent';

export function Chat() {
  const { messages } = useChat<MyAgentUIMessage>();
  // 메시지와 도구에 대한 완전한 타입 안전성
}
```
