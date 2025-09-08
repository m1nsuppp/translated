# Tool Calling (툴 호출)

Foundations에서 다뤘듯이, [tools](/docs/foundations/tools)는 모델이 특정 작업을 수행하기 위해 호출할 수 있는 객체야.

AI SDK Core의 툴은 세 가지 요소로 구성돼:

- **`description`**: 선택적 설명. 툴이 언제 선택될지에 영향을 줄 수 있어.
- **`inputSchema`**: 입력 파라미터를 정의하는 [Zod 스키마](/docs/foundations/tools#schemas) 또는 [JSON 스키마](/docs/reference/ai-sdk-core/json-schema). 이 스키마는 LLM이 소비하고, LLM의 툴 호출을 검증하는 데도 사용돼.
- **`execute`**: 툴 호출의 입력을 받아 실행되는 선택적 async 함수. `RESULT`(제네릭) 타입 값을 반환해. 같은 프로세스에서 실행하지 않고 클라이언트나 큐로 포워딩하려면 생략할 수도 있어.

<Note className="mb-2">
  [`tool`](/docs/reference/ai-sdk-core/tool) 헬퍼로 `execute` 파라미터 타입을 추론할 수 있어.
</Note>

`generateText`와 `streamText`의 `tools` 파라미터는 툴 이름을 키로, 툴 객체를 값으로 가지는 객체야:

```ts highlight="6-17"
import { z } from 'zod';
import { generateText, tool } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      inputSchema: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
  },
  prompt: 'What is the weather in San Francisco?',
});
```

<Note>
  모델이 툴을 사용할 때를 "tool call"이라고 하고, 툴의 출력을 "tool result"라고 해.
</Note>

툴 호출은 텍스트 생성에만 제한되지 않아. 생성형 UI(Generative UI) 렌더링에도 쓸 수 있어.

## Multi-Step Calls (using stopWhen)

`stopWhen` 설정을 사용하면 `generateText`와 `streamText`에서 멀티 스텝 호출을 활성화할 수 있어. `stopWhen`이 설정되어 있고 모델이 툴 호출을 생성하면, AI SDK는 더 이상 툴 호출이 없거나 중지 조건을 만족할 때까지 툴 결과를 넘겨주며 새로운 생성을 이어가.

<Note>
  `stopWhen` 조건은 마지막 스텝에 툴 결과가 포함된 경우에만 평가돼.
</Note>

기본적으로 `generateText`나 `streamText`는 단일 생성을 트리거해. 많은 경우 이걸로 충분하지만, 툴을 제공하면 모델은 일반 텍스트 응답을 생성하거나 툴 호출을 생성할 수 있어. 모델이 툴 호출을 생성하면 그 생성은 해당 스텝에서 종료돼.

툴 실행 후에 모델이 그 결과를 요약하거나 사용자 질의 컨텍스트에 맞게 텍스트를 생성하길 원할 수도 있어. 또한 하나의 응답에서 여러 툴을 쓰게 하고 싶을 수도 있지. 이럴 때 멀티 스텝 호출이 필요해.

사람과의 대화에 비유하면 쉬워. 질문을 받았을 때, 상대가 일반 지식으로 답할 수 없으면 정보를 찾아본 뒤(툴 사용) 답하잖아. 모델도 같은 방식으로 필요한 정보를 얻기 위해 툴을 호출한 다음 답할 수 있어. 각 생성(툴 호출 또는 텍스트 생성)이 하나의 스텝이야.

### 예시

아래 예시는 두 개의 스텝으로 진행돼:

1. **Step 1**
   1. 프롬프트 `'What is the weather in San Francisco?'`가 모델에 전송됨.
   1. 모델이 툴 호출을 생성.
   1. 툴 호출이 실행됨.
1. **Step 2**
   1. 툴 결과가 모델에 전달됨.
   1. 모델이 툴 결과를 고려해 응답을 생성.

```ts highlight="18-19"
import { z } from 'zod';
import { generateText, tool, stepCountIs } from 'ai';

const { text, steps } = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      inputSchema: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
  },
  stopWhen: stepCountIs(5), // 툴이 호출됐다면 최대 5 스텝에서 중지
  prompt: 'What is the weather in San Francisco?',
});
```

<Note>`streamText`도 유사하게 사용할 수 있어.</Note>

### Steps

중간 툴 호출과 결과에 접근하려면 결과 객체의 `steps` 프로퍼티를 사용하거나, `streamText`의 `onFinish` 콜백을 사용해.
각 스텝의 텍스트, 툴 호출, 툴 결과 등 모든 정보를 포함해.

#### 예시: 모든 스텝에서 툴 결과 추출하기

```ts highlight="3,9-10"
import { generateText } from 'ai';

const { steps } = await generateText({
  model: openai('gpt-4o'),
  stopWhen: stepCountIs(10),
  // ...
});

// extract all tool calls from the steps:
const allToolCalls = steps.flatMap(step => step.toolCalls);
```

### `onStepFinish` 콜백

`generateText`나 `streamText`를 사용할 때, 스텝이 끝나면 호출되는 `onStepFinish` 콜백을 제공할 수 있어.
즉, 해당 스텝의 모든 텍스트 델타, 툴 호출, 툴 결과가 준비되면 호출돼. 멀티 스텝이면 스텝마다 한 번씩 호출돼.

```tsx highlight="5-7"
import { generateText } from 'ai';

const result = await generateText({
  // ...
  onStepFinish({ text, toolCalls, toolResults, finishReason, usage }) {
    // 예: 대화 기록 저장, 사용량 기록 등
  },
});
```

### `prepareStep` 콜백

`prepareStep` 콜백은 스텝 시작 전에 호출돼.

다음 파라미터를 받아:

- `model`: `generateText`에 전달한 모델
- `stopWhen`: `generateText`에 전달한 중지 조건
- `stepNumber`: 현재 실행 중인 스텝 번호
- `steps`: 지금까지 실행된 스텝들
- `messages`: 현재 스텝에서 모델로 보낼 메시지들

이 콜백으로 스텝별로 다른 설정을 제공하거나 입력 메시지를 수정할 수 있어.

```tsx highlight="5-7"
import { generateText } from 'ai';

const result = await generateText({
  // ...
  prepareStep: async ({ model, stepNumber, steps, messages }) => {
    if (stepNumber === 0) {
      return {
        // 이 스텝만 다른 모델 사용
        model: modelForThisParticularStep,
        // 이 스텝에서 특정 툴 사용 강제
        toolChoice: { type: 'tool', toolName: 'tool1' },
        // 이 스텝에서 사용 가능한 툴 제한
        activeTools: ['tool1'],
      };
    }

    // 아무 것도 반환하지 않으면 기본 설정 사용
  },
});
```

#### 긴 에이전틱 루프를 위한 메시지 수정

긴 루프에서 각 스텝의 입력 메시지를 `messages`로 수정할 수 있어. 특히 프롬프트 압축에 유용해:

```tsx
prepareStep: async ({ stepNumber, steps, messages }) => {
  // 히스토리가 너무 길면 최근만 유지
  if (messages.length > 20) {
    return {
      messages: messages.slice(-10),
    };
  }

  return {};
},
```

## Response Messages

멀티 스텝 툴 호출을 쓰면, 생성된 assistant/tool 메시지를 대화 역사에 추가하는 일이 흔해.

`generateText`와 `streamText` 모두 `response.messages`를 제공하니까, 이걸 대화 기록에 추가하면 돼.
`streamText`의 `onFinish`에서도 사용 가능해.

`response.messages`는 대화 히스토리에 추가할 수 있는 `ModelMessage` 배열이야:

```ts
import { generateText, ModelMessage } from 'ai';

const messages: ModelMessage[] = [
  // ...
];

const { response } = await generateText({
  // ...
  messages,
});

// 응답 메시지를 대화 기록에 추가
messages.push(...response.messages); // streamText: ...((await response).messages)
```

## Dynamic Tools

컴파일 타임에 툴 스키마를 알 수 없는 상황을 위해 동적 툴을 지원해. 다음과 같은 경우 유용해:

- 스키마 없는 MCP(Model Context Protocol) 툴
- 런타임 사용자 정의 함수
- 외부 소스에서 로드된 툴

### `dynamicTool` 사용하기

`dynamicTool` 헬퍼는 입력/출력 타입이 미확정인 툴을 만들어:

```ts
import { dynamicTool } from 'ai';
import { z } from 'zod';

const customTool = dynamicTool({
  description: 'Execute a custom function',
  inputSchema: z.object({}),
  execute: async input => {
    // input 타입은 'unknown'
    // 런타임에 검증/캐스팅 필요
    const { action, parameters } = input as any;

    // 커스텀 로직 실행
    return { result: `Executed ${action}` };
  },
});
```

### 타입 안전 처리

정적/동적 툴을 함께 쓸 때는 `dynamic` 플래그로 타입 내로잉을 해:

```ts
const result = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    // 정적 툴(타입 확정)
    weather: weatherTool,
    // 동적 툴
    custom: dynamicTool({
      /* ... */
    }),
  },
  onStepFinish: ({ toolCalls, toolResults }) => {
    // 타입 안전한 순회
    for (const toolCall of toolCalls) {
      if (toolCall.dynamic) {
        // 동적 툴: input은 'unknown'
        console.log('Dynamic:', toolCall.toolName, toolCall.input);
        continue;
      }

      // 정적 툴: 완전한 타입 추론 가능
      switch (toolCall.toolName) {
        case 'weather':
          console.log(toolCall.input.location); // string으로 타이핑됨
          break;
      }
    }
  },
});
```

## Preliminary Tool Results

여러 결과를 내보내는 `AsyncIterable`을 반환할 수 있어. 이 경우 이터러블의 마지막 값이 최종 툴 결과야.

툴 실행 중 상태 정보를 스트리밍하려는 경우(예: 제너레이터 함수와 조합) 유용해:

```ts
tool({
  description: 'Get the current weather.',
  inputSchema: z.object({
    location: z.string(),
  }),
  async *execute({ location }) {
    yield {
      status: 'loading' as const,
      text: `Getting weather for ${location}`,
      weather: undefined,
    };

    await new Promise(resolve => setTimeout(resolve, 3000));

    const temperature = 72 + Math.floor(Math.random() * 21) - 10;

    yield {
      status: 'success' as const,
      text: `The weather in ${location} is ${temperature}°F`,
      temperature,
    };
  },
});
```

## Tool Choice

`toolChoice`로 툴 선택 시점을 제어할 수 있어. 다음 설정을 지원해:

- `auto`(기본): 모델이 툴 호출 여부와 대상을 선택
- `required`: 반드시 툴을 호출(어떤 툴인지는 모델이 선택)
- `none`: 툴을 호출하지 않음
- `{ type: 'tool', toolName: string }`: 지정된 툴을 반드시 호출

```ts highlight="18"
import { z } from 'zod';
import { generateText, tool } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4o',
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      inputSchema: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
  },
  toolChoice: 'required', // 툴 호출 강제
  prompt: 'What is the weather in San Francisco?',
});
```

## Tool Execution Options

툴이 호출되면, 두 번째 인자로 추가 옵션을 받아.

### Tool Call ID

툴 호출 ID가 실행에 전달돼. 스트림 데이터와 함께 툴 호출 관련 정보를 보낼 때 활용 가능해.

```ts highlight="14-20"
import {
  streamText,
  tool,
  createUIMessageStream,
  createUIMessageStreamResponse,
} from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const stream = createUIMessageStream({
    execute: ({ writer }) => {
      const result = streamText({
        // ...
        messages,
        tools: {
          myTool: tool({
            // ...
            execute: async (args, { toolCallId }) => {
              // 예: 툴 호출 상태를 커스텀 전송
              writer.write({
                type: 'data-tool-status',
                id: toolCallId,
                data: {
                  name: 'myTool',
                  status: 'in-progress',
                },
              });
              // ...
            },
          }),
        },
      });

      writer.merge(result.toUIMessageStream());
    },
  });

  return createUIMessageStreamResponse({ stream });
}
```

### Messages

툴 호출이 포함된 응답을 시작했던 LLM 입력 메시지들이 툴 실행의 두 번째 인자로 전달돼. 멀티 스텝에서는 이전 스텝까지의 텍스트/툴 호출/툴 결과가 모두 포함돼.

```ts highlight="8-9"
import { generateText, tool } from 'ai';

const result = await generateText({
  // ...
  tools: {
    myTool: tool({
      // ...
      execute: async (args, { messages }) => {
        // 예: 다른 LLM 호출 시 히스토리 활용
        return { ... };
      },
    }),
  },
});
```

### Abort Signals

`generateText`와 `streamText`의 abort signal이 툴 실행으로 전달돼. 툴 내부에서 오래 걸리는 연산을 중단하거나 `fetch`에 전달할 수 있어.

```ts highlight="6,11,14"
import { z } from 'zod';
import { generateText, tool } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4.1',
  abortSignal: myAbortSignal, // 툴로 전달될 시그널
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      inputSchema: z.object({ location: z.string() }),
      execute: async ({ location }, { abortSignal }) => {
        return fetch(
          `https://api.weatherapi.com/v1/current.json?q=${location}`,
          { signal: abortSignal }, // fetch에 시그널 전달
        );
      },
    }),
  },
  prompt: 'What is the weather in San Francisco?',
});
```

### Context (experimental)

`experimental_context` 설정으로 임의의 컨텍스트를 전달할 수 있어. 툴 실행 옵션의 `experimental_context`에서 접근 가능해.

```ts
const result = await generateText({
  // ...
  tools: {
    someTool: tool({
      // ...
      execute: async (input, { experimental_context: context }) => {
        const typedContext = context as { example: string }; // 또는 타입 검증 라이브러리 사용
        // ...
      },
    }),
  },
  experimental_context: { example: '123' },
});
```

## Types

모듈화된 코드에서는 타입을 정의해 재사용성과 타입 안전성을 보장하는 게 좋아. 이를 위해 AI SDK는 툴, 툴 호출, 툴 결과용 헬퍼 타입을 제공해.

이 타입들로 `streamText`/`generateText` 직접 영역 밖의 변수/함수 파라미터/리턴 타입을 강하게 타이핑할 수 있어.

각 툴 호출은 호출된 툴에 따라 `ToolCall<NAME extends string, ARGS>`로 타이핑돼.
툴 결과도 마찬가지로 `ToolResult<NAME extends string, ARGS, RESULT>`로 타이핑돼.

`streamText`와 `generateText`의 툴 집합은 `ToolSet`으로 정의돼.
타입 추론 헬퍼인 `TypedToolCall<TOOLS extends ToolSet>`과 `TypedToolResult<TOOLS extends ToolSet>`로 툴 호출/결과 타입을 추출할 수 있어.

```ts highlight="18-19,23-24"
import { openai } from '@ai-sdk/openai';
import { TypedToolCall, TypedToolResult, generateText, tool } from 'ai';
import { z } from 'zod';

const myToolSet = {
  firstTool: tool({
    description: 'Greets the user',
    inputSchema: z.object({ name: z.string() }),
    execute: async ({ name }) => `Hello, ${name}!`,
  }),
  secondTool: tool({
    description: 'Tells the user their age',
    inputSchema: z.object({ age: z.number() }),
    execute: async ({ age }) => `You are ${age} years old!`,
  }),
};

type MyToolCall = TypedToolCall<typeof myToolSet>;
type MyToolResult = TypedToolResult<typeof myToolSet>;

async function generateSomething(prompt: string): Promise<{
  text: string;
  toolCalls: Array<MyToolCall>; // typed tool calls
  toolResults: Array<MyToolResult>; // typed tool results
}> {
  return generateText({
    model: openai('gpt-4.1'),
    tools: myToolSet,
    prompt,
  });
}
```

## Handling Errors

AI SDK에는 툴 호출 관련 에러가 세 가지 있어:

- [`NoSuchToolError`](/docs/reference/ai-sdk-errors/ai-no-such-tool-error): tools 객체에 정의되지 않은 툴을 모델이 호출한 경우
- [`InvalidToolInputError`](/docs/reference/ai-sdk-errors/ai-invalid-tool-input-error): 툴 입력이 해당 툴의 입력 스키마와 맞지 않는 경우
- [`ToolCallRepairError`](/docs/reference/ai-sdk-errors/ai-tool-call-repair-error): 툴 호출 복구 도중 발생한 에러

툴 실행이 실패하면(네 `execute`에서 에러 throw), AI SDK는 멀티 스텝 자동 라운드트립을 위해 `tool-error` 콘텐츠 파트로 추가해.

### `generateText`

`generateText`는 툴 스키마 검증 이슈 및 기타 에러를 throw하므로 `try`/`catch`로 핸들링하면 돼. 툴 실행 에러는 결과 스텝들에 `tool-error` 파트로 나타나.

```ts
try {
  const result = await generateText({
    //...
  });
} catch (error) {
  if (NoSuchToolError.isInstance(error)) {
    // 없는 툴 호출 처리
  } else if (InvalidToolInputError.isInstance(error)) {
    // 잘못된 입력 처리
  } else {
    // 기타 에러 처리
  }
}
```

툴 실행 에러는 스텝에서 확인할 수 있어:

```ts
const { steps } = await generateText({
  // ...
});

// 스텝들에서 tool-error 파트 찾기
const toolErrors = steps.flatMap(step =>
  step.content.filter(part => part.type === 'tool-error'),
);

toolErrors.forEach(toolError => {
  console.log('Tool error:', toolError.error);
  console.log('Tool name:', toolError.toolName);
  console.log('Tool input:', toolError.input);
});
```

### `streamText`

`streamText`는 에러를 전체 스트림의 일부로 보냄. 툴 실행 에러는 `tool-error`, 기타 에러는 `error` 파트로 전달돼.

`toUIMessageStreamResponse` 사용 시 `onError`로 에러 메시지를 추출해 스트림 응답에 포함시킬 수 있어:

```ts
const result = streamText({
  // ...
});

return result.toUIMessageStreamResponse({
  onError: error => {
    if (NoSuchToolError.isInstance(error)) {
      return 'The model tried to call a unknown tool.';
    } else if (InvalidToolInputError.isInstance(error)) {
      return 'The model called a tool with invalid inputs.';
    } else {
      return 'An unknown error occurred.';
    }
  },
});
```

## Tool Call Repair

<Note type="warning">
  툴 호출 복구 기능은 실험적이며 향후 변경될 수 있어.
</Note>

특히 입력 스키마가 복잡하거나 모델이 작은 경우, LLM이 유효한 툴 호출을 생성하지 못할 때가 있어.

멀티 스텝을 사용하면, 실패한 툴 호출을 다음 스텝에서 다시 모델에 보내 수정 기회를 줄 수 있어. 하지만 추가 스텝 없이, 메시지 히스토리를 오염시키지 않고도 복구를 제어하고 싶을 수 있지.

`experimental_repairToolCall`로 커스텀 함수 기반 복구를 시도할 수 있어.

복구 전략 예:

- 구조화 출력이 가능한 모델로 입력 생성
- 메시지/시스템 프롬프트/툴 스키마를 더 강한 모델에 보내 입력 생성
- 호출된 툴에 따라 더 구체적인 복구 지시 제공

### 예: 구조화 출력 모델로 복구

```ts
import { openai } from '@ai-sdk/openai';
import { generateObject, generateText, NoSuchToolError, tool } from 'ai';

const result = await generateText({
  model,
  tools,
  prompt,

  experimental_repairToolCall: async ({
    toolCall,
    tools,
    inputSchema,
    error,
  }) => {
    if (NoSuchToolError.isInstance(error)) {
      return null; // 잘못된 툴 이름은 복구 시도 안 함
    }

    const tool = tools[toolCall.toolName as keyof typeof tools];

    const { object: repairedArgs } = await generateObject({
      model: openai('gpt-4.1'),
      schema: tool.inputSchema,
      prompt: [
        `The model tried to call the tool "${toolCall.toolName}"` +
          ` with the following inputs:`,
        JSON.stringify(toolCall.input),
        `The tool accepts the following schema:`,
        JSON.stringify(inputSchema(toolCall)),
        'Please fix the inputs.',
      ].join('\n'),
    });

    return { ...toolCall, input: JSON.stringify(repairedArgs) };
  },
});
```

### 예: 재질문(re-ask) 전략으로 복구

```ts
import { openai } from '@ai-sdk/openai';
import { generateObject, generateText, NoSuchToolError, tool } from 'ai';

const result = await generateText({
  model,
  tools,
  prompt,

  experimental_repairToolCall: async ({
    toolCall,
    tools,
    error,
    messages,
    system,
  }) => {
    const result = await generateText({
      model,
      system,
      messages: [
        ...messages,
        {
          role: 'assistant',
          content: [
            {
              type: 'tool-call',
              toolCallId: toolCall.toolCallId,
              toolName: toolCall.toolName,
              input: toolCall.input,
            },
          ],
        },
        {
          role: 'tool' as const,
          content: [
            {
              type: 'tool-result',
              toolCallId: toolCall.toolCallId,
              toolName: toolCall.toolName,
              output: error.message,
            },
          ],
        },
      ],
      tools,
    });

    const newToolCall = result.toolCalls.find(
      newToolCall => newToolCall.toolName === toolCall.toolName,
    );

    return newToolCall != null
      ? {
          toolCallType: 'function' as const,
          toolCallId: toolCall.toolCallId,
          toolName: toolCall.toolName,
          input: JSON.stringify(newToolCall.input),
        }
      : null;
  },
});
```

## Active Tools

모델에 따라 한 번에 처리할 수 있는 툴 수가 제한돼. 많은 툴을 정적으로 타입 유지하면서도, 모델에 제공되는 툴을 동시에 제한하기 위해 `activeTools` 프로퍼티를 제공해.

현재 활성화된 툴 이름 배열이야. 기본값은 `undefined`이며 이 경우 모든 툴이 활성화돼.

```ts highlight="7"
import { openai } from '@ai-sdk/openai';
import { generateText } from 'ai';

const { text } = await generateText({
  model: openai('gpt-4.1'),
  tools: myToolSet,
  activeTools: ['firstTool'],
});
```

## Multi-modal Tool Results

<Note type="warning">
  멀티모달 툴 결과는 실험적이며 Anthropic만 지원해.
</Note>

스크린샷 같은 멀티모달 툴 결과를 모델로 다시 보내려면, 특정 포맷으로 변환해야 해.

AI SDK Core 툴은 결과를 콘텐츠 파트로 변환하는 선택적 `toModelOutput` 함수를 제공해.

아래는 스크린샷을 콘텐츠 파트로 변환하는 예시야:

```ts highlight="22-27"
const result = await generateText({
  model: anthropic('claude-3-5-sonnet-20241022'),
  tools: {
    computer: anthropic.tools.computer_20241022({
      // ...
      async execute({ action, coordinate, text }) {
        switch (action) {
          case 'screenshot': {
            return {
              type: 'image',
              data: fs
                .readFileSync('./data/screenshot-editor.png')
                .toString('base64'),
            };
          }
          default: {
            return `executed ${action}`;
          }
        }
      },

      // LLM이 소비할 툴 결과 콘텐츠로 매핑
      toModelOutput(result) {
        return {
          type: 'content',
          value:
            typeof result === 'string'
              ? [{ type: 'text', text: result }]
              : [{ type: 'image', data: result.data, mediaType: 'image/png' }],
        };
      },
    }),
  },
  // ...
});
```

## Extracting Tools

툴이 많아지면 개별 파일로 분리하고 싶어질 거야. `tool` 헬퍼는 타입 추론을 보장하므로 중요해.

분리한 툴 예시는 다음과 같아:

```ts filename="tools/weather-tool.ts" highlight="1,4-5"
import { tool } from 'ai';
import { z } from 'zod';

// `tool` 헬퍼가 타입 추론을 보장
export const weatherTool = tool({
  description: 'Get the weather in a location',
  inputSchema: z.object({
    location: z.string().describe('The location to get the weather for'),
  }),
  execute: async ({ location }) => ({
    location,
    temperature: 72 + Math.floor(Math.random() * 21) - 10,
  }),
});
```

## MCP Tools

<Note type="warning">
  MCP 툴 기능은 실험적이며 향후 변경될 수 있어.
</Note>

AI SDK는 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 서버에 연결해 해당 툴에 접근하는 걸 지원해. 이를 통해 다양한 서비스의 툴을 표준 인터페이스로 발견하고 사용할 수 있어.

### MCP 클라이언트 초기화

다음 중 하나로 MCP 클라이언트를 생성할 수 있어:

- `SSE`(Server-Sent Events): HTTP 기반 실시간 통신. 네트워크越 데이터 전송이 필요한 원격 서버에 적합
- `stdio`: 표준 입출력 스트림 통신. 같은 머신에서 실행되는 로컬 툴 서버(예: CLI 툴, 로컬 서비스)에 적합
- 커스텀 트랜스포트: `MCPTransport` 인터페이스를 구현해 직접 제공. MCP 공식 TypeScript SDK의 `StreamableHTTPClientTransport` 등 사용 시 적합

#### SSE Transport

SSE는 `type`과 `url`을 가진 단순 객체로 설정할 수 있어:

```typescript
import { experimental_createMCPClient as createMCPClient } from 'ai';

const mcpClient = await createMCPClient({
  transport: {
    type: 'sse',
    url: 'https://my-server.com/sse',

    // 선택: 인증 등 HTTP 헤더 설정
    headers: {
      Authorization: 'Bearer my-api-key',
    },
  },
});
```

#### Stdio Transport

Stdio 트랜스포트는 `ai/mcp-stdio` 패키지의 `StdioMCPTransport` 클래스를 임포트해야 해:

```typescript
import { experimental_createMCPClient as createMCPClient } from 'ai';
import { Experimental_StdioMCPTransport as StdioMCPTransport } from 'ai/mcp-stdio';

const mcpClient = await createMCPClient({
  transport: new StdioMCPTransport({
    command: 'node',
    args: ['src/stdio/dist/server.js'],
  }),
});
```

#### Custom Transport

`MCPTransport` 인터페이스만 구현하면 직접 트랜스포트를 제공할 수 있어. 아래는 MCP 공식 TS SDK의 `StreamableHTTPClientTransport`를 사용하는 예시야:

```typescript
import {
  MCPTransport,
  experimental_createMCPClient as createMCPClient,
} from 'ai';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp';

const url = new URL('http://localhost:3000/mcp');
const mcpClient = await createMCPClient({
  transport: new StreamableHTTPClientTransport(url, {
    sessionId: 'session_123',
  }),
});
```

<Note>
  `experimental_createMCPClient`가 반환하는 클라이언트는 툴 변환 용도의 경량 클라이언트야. 현재는 풀 MCP 클라이언트의 모든 기능(인증, 세션 관리, 재개 가능한 스트림, 알림 수신 등)을 지원하지 않아.
</Note>

#### MCP 클라이언트 종료

사용 패턴에 따라 MCP 클라이언트를 적절히 종료해야 해:

- 단발성 사용(단일 요청 등): 응답이 끝나면 클라이언트 종료
- 장수명 클라이언트(CLI 앱 등): 실행 유지하되 앱 종료 시 닫도록 보장

스트리밍 응답을 사용할 때는 LLM 응답이 끝났을 때 클라이언트를 닫아. 예를 들어 `streamText`에서는 `onFinish`를 사용해:

```typescript
const mcpClient = await experimental_createMCPClient({
  // ...
});

const tools = await mcpClient.tools();

const result = await streamText({
  model: openai('gpt-4.1'),
  tools,
  prompt: 'What is the weather in Brooklyn, New York?',
  onFinish: async () => {
    await mcpClient.close();
  },
});
```

스트리밍 없이 생성할 땐, try/finally나 프레임워크의 클린업 훅을 써:

```typescript
let mcpClient: MCPClient | undefined;

try {
  mcpClient = await experimental_createMCPClient({
    // ...
  });
} finally {
  await mcpClient?.close();
}
```

### MCP 툴 사용하기

클라이언트의 `tools` 메서드는 MCP 툴과 AI SDK 툴 사이의 어댑터 역할을 해. 툴 스키마를 다루는 두 가지 접근을 지원해:

#### 스키마 디스커버리

서버가 제공하는 모든 툴을 나열하고, 서버 스키마 기반으로 입력 파라미터 타입을 추론하는 가장 단순한 접근:

```typescript
const tools = await mcpClient.tools();
```

**장점:**

- 구현이 단순
- 서버 변경에 자동으로 동기화

**단점:**

- 개발 중 TypeScript 타입 안전성 부족
- 툴 파라미터 IDE 자동완성 미지원
- 오류가 런타임에만 드러남
- 서버의 모든 툴을 로드함

#### 스키마 정의

클라이언트 코드에서 툴과 입력 스키마를 명시적으로 정의하는 접근:

```typescript
import { z } from 'zod';

const tools = await mcpClient.tools({
  schemas: {
    'get-data': {
      inputSchema: z.object({
        query: z.string().describe('The data query'),
        format: z.enum(['json', 'text']).optional(),
      }),
    },
    // 입력이 없는 툴은 빈 객체 스키마 사용
    'tool-with-no-args': {
      inputSchema: z.object({}),
    },
  },
});
```

**장점:**

- 로드할 툴을 통제
- 완전한 타입 안전성
- IDE 자동완성 향상
- 개발 중 파라미터 불일치 조기 발견

**단점:**

- 서버 스키마와 동기화를 수동으로 유지해야 함
- 관리할 코드 증가

`schemas`를 정의하면 서버가 더 많은 툴을 제공해도 명시한 툴만 불러와. 다음에 유리해:

- 애플리케이션을 필요한 툴에만 집중
- 불필요한 툴 로딩 감소
- 툴 의존성을 명시적으로 유지
