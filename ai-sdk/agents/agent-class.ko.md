# Agent 클래스

Agent 클래스는 LLM 설정, 도구, 동작을 재사용 가능한 컴포넌트로 캡슐화하는 구조적 방법을 제공해. 에이전트 루프를 대신 처리해 주기 때문에, LLM이 복잡한 작업을 완료하기 위해 여러 번 연속으로 도구를 호출할 수 있어. 한 번 정의해두면 앱 전반에서 재사용할 수 있어.

## Agent 클래스를 왜 쓰나요?

AI 애플리케이션을 만들 때 보통 다음이 필요해:

- **설정 재사용**: 동일한 모델 설정, 도구, 프롬프트를 앱 곳곳에서 재사용
- **일관성 유지**: 코드베이스 전반에서 동일한 동작과 능력을 보장
- **API 라우트 단순화**: 엔드포인트 보일러플레이트를 줄임
- **타입 안전성**: 에이전트의 도구와 출력에 대해 완전한 TypeScript 지원

Agent 클래스는 에이전트의 행동을 정의하는 단일한 위치를 제공해.

## 에이전트 생성

원하는 구성으로 Agent 클래스를 인스턴스화해서 에이전트를 정의해:

```ts
import { Experimental_Agent as Agent } from 'ai';

const myAgent = new Agent({
  model: 'openai/gpt-4o',
  system: 'You are a helpful assistant.',
  tools: {
    // Your tools here
  },
});
```

## 구성 옵션

Agent 클래스는 `generateText`와 `streamText`와 동일한 설정을 모두 받아. 설정 항목:

### Model과 System 프롬프트

```ts
import { Experimental_Agent as Agent } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  system: 'You are an expert software engineer.',
});
```

### Tools

에이전트가 작업 수행에 사용할 도구를 제공해:

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
        // Execute code and return result
        return { output: 'Code executed successfully' };
      },
    }),
  },
});
```

### 루프 제어

기본적으로 에이전트는 한 단계만 실행돼. 각 단계에서 모델은 텍스트를 생성하거나 도구를 호출해. 텍스트를 생성하면 완료되고, 도구를 호출하면 AI SDK가 그 도구를 실행해.

여러 도구를 연속으로 호출하려면 `stopWhen`을 설정해 단계 수를 늘려. 각 도구 실행 후 모델은 새 생성을 트리거하고, 그때 다시 도구를 호출하거나 텍스트를 생성할 수 있어:

```ts
import { Experimental_Agent as Agent, stepCountIs } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  stopWhen: stepCountIs(10), // 최대 10단계 허용
});
```

각 단계는 하나의 생성(텍스트 또는 도구 호출)이며, 다음 조건 중 하나가 될 때까지 루프가 계속돼:

- 모델이 도구 호출 대신 텍스트를 생성하거나,
- 정지 조건이 만족되거나

복수 조건을 조합할 수도 있어:

```ts
import { Experimental_Agent as Agent, stepCountIs } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  stopWhen: [
    stepCountIs(10), // 최대 10단계
    yourCustomCondition(), // 커스텀 정지 로직
  ],
});
```

더 알아보기: [루프 제어와 정지 조건](/docs/agents/loop-control).

### Tool Choice

에이전트의 도구 사용 방식을 제어해:

```ts
import { Experimental_Agent as Agent } from 'ai';

const agent = new Agent({
  model: 'openai/gpt-4o',
  tools: {
    // your tools here
  },
  toolChoice: 'required', // 도구 사용 강제
  // 또는 toolChoice: 'none' 으로 도구 비활성화
  // 또는 toolChoice: 'auto' (기본) 로 모델에게 맡김
});
```

특정 도구 사용을 강제할 수도 있어:

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
    toolName: 'weather', // weather 도구 강제 사용
  },
});
```

### 구조화된 출력

구조화된 출력 스키마를 정의해:

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

## 시스템 프롬프트로 에이전트 행동 정의

시스템 프롬프트는 에이전트의 행동, 성격, 제약을 정의해. 모든 상호작용의 컨텍스트를 세팅하고, 에이전트가 사용자 요청에 응답하고 도구를 사용하는 방식을 안내해.

### 기본 System 프롬프트

에이전트의 역할과 전문성을 설정해:

```ts
const agent = new Agent({
  model: 'openai/gpt-4o',
  system: 'You are an expert data analyst. You provide clear insights from complex data.',
});
```

### 상세 행동 지침

에이전트 행동에 대한 구체 지침을 제공해:

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

### 행동 제약

경계를 설정하고 일관된 행동을 보장해:

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

가용 도구 사용 방식을 안내해:

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

### 출력 형식과 스타일 지침

출력 포맷과 커뮤니케이션 스타일을 제어해:

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

## 에이전트 사용하기

정의한 에이전트는 세 가지 방식으로 사용할 수 있어:

### 텍스트 생성

일회성 텍스트 생성에는 `generate()`를 사용해:

```ts
const result = await myAgent.generate({
  prompt: 'What is the weather like?',
});

console.log(result.text);
```

### 스트리밍 텍스트

스트리밍 응답에는 `stream()`을 사용해:

```ts
const stream = myAgent.stream({
  prompt: 'Tell me a story',
});

for await (const chunk of stream.textStream) {
  console.log(chunk);
}
```

### UI 메시지에 응답

클라이언트 앱을 위한 API 응답에는 `respond()`를 사용해:

```ts
// In your API route (e.g., app/api/chat/route.ts)
import { validateUIMessages } from 'ai';

export async function POST(request: Request) {
  const { messages } = await request.json();

  return myAgent.respond({
    messages: await validateUIMessages({ messages }),
  });
}
```

## End-to-end 타입 안전성

에이전트의 `UIMessage` 타입을 추론할 수 있어:

```ts
import {
  Experimental_Agent as Agent,
  Experimental_InferAgentUIMessage as InferAgentUIMessage,
} from 'ai';

const myAgent = new Agent({
  // ... configuration
});

// Infer the UIMessage type for UI components or persistence
export type MyAgentUIMessage = InferAgentUIMessage<typeof myAgent>;
```

클라이언트 컴포넌트에서 `useChat`과 함께 타입을 활용해:

```tsx filename="components/chat.tsx"
'use client';

import { useChat } from '@ai-sdk/react';
import type { MyAgentUIMessage } from '@/agent/my-agent';

export function Chat() {
  const { messages } = useChat<MyAgentUIMessage>();
  // 메시지와 도구에 대한 완전한 타입 안전성
}
```
