# Chatbot Tool 사용법

[`useChat`](/docs/reference/ai-sdk-ui/use-chat)와 [`streamText`](/docs/reference/ai-sdk-core/stream-text)를 사용하면 챗봇 앱에서 툴을 활용할 수 있어.
AI SDK는 이 컨텍스트에서 아래 세 가지 타입의 툴을 지원해:

1. 서버에서 자동 실행되는 툴
2. 클라이언트에서 자동 실행되는 툴
3. 확인 다이얼로그 같은 사용자 상호작용이 필요한 툴

플로우는 다음과 같아:

1. 사용자가 채팅 UI에 메시지를 입력해.
2. 메시지가 API 라우트로 전송돼.
3. 서버 라우트에서 `streamText` 호출 중에 LLM이 툴 호출을 생성해.
4. 생성된 모든 툴 호출은 클라이언트로 전달돼.
5. 서버 사이드 툴은 `execute` 메서드로 실행되고, 그 결과가 클라이언트로 전달돼.
6. 클라이언트에서 자동 실행되는 툴은 `onToolCall` 콜백으로 처리해.
   결과를 제공하려면 반드시 `addToolResult`를 호출해야 해.
7. 사용자 상호작용이 필요한 클라이언트 툴은 UI에 표시할 수 있어.
   툴 호출과 결과는 마지막 어시스턴트 메시지의 `parts` 속성에 있는 툴 인보케이션 파트로 접근 가능해.
8. 사용자 상호작용이 끝나면, `addToolResult`로 툴 결과를 채팅에 추가할 수 있어.
9. 모든 툴 결과가 준비되면 자동 제출되도록 `sendAutomaticallyWhen`을 설정할 수 있어.
   이렇게 하면 이 플로우가 한 번 더 반복돼.

툴 호출과 실행은 어시스턴트 메시지 안의 타입이 지정된 툴 파트로 통합돼.
툴 파트는 처음엔 툴 호출로 시작하고, 실행되면 툴 결과로 전환돼.
툴 결과에는 툴 호출 정보와 실행 결과가 모두 포함돼.

<Note>
  `sendAutomaticallyWhen` 옵션으로 툴 결과 전송 방식을 설정할 수 있어.
  `lastAssistantMessageIsCompleteWithToolCalls` 헬퍼를 사용하면 모든 툴 결과가 준비됐을 때 자동으로 제출할 수 있어.
  이렇게 하면 필요한 경우의 완전한 제어는 유지하면서도 클라이언트 코드가 더 단순해져.
</Note>

## 예시

이 예시에서는 세 가지 툴을 사용해:

- `getWeatherInformation`: 특정 도시의 날씨를 반환하는 서버 자동 실행 툴
- `askForConfirmation`: 사용자 확인을 받는 클라이언트 상호작용 툴
- `getLocation`: 랜덤 도시를 반환하는 클라이언트 자동 실행 툴

### API 라우트

```tsx filename='app/api/chat/route.ts'
import { openai } from '@ai-sdk/openai';
import { convertToModelMessages, streamText, UIMessage } from 'ai';
import { z } from 'zod';

// Allow streaming responses up to 30 seconds
export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages: convertToModelMessages(messages),
    tools: {
      // server-side tool with execute function:
      getWeatherInformation: {
        description: 'show the weather in a given city to the user',
        inputSchema: z.object({ city: z.string() }),
        execute: async ({}: { city: string }) => {
          const weatherOptions = ['sunny', 'cloudy', 'rainy', 'snowy', 'windy'];
          return weatherOptions[
            Math.floor(Math.random() * weatherOptions.length)
          ];
        },
      },
      // client-side tool that starts user interaction:
      askForConfirmation: {
        description: 'Ask the user for confirmation.',
        inputSchema: z.object({
          message: z.string().describe('The message to ask for confirmation.'),
        }),
      },
      // client-side tool that is automatically executed on the client:
      getLocation: {
        description:
          'Get the user location. Always ask for confirmation before using this tool.',
        inputSchema: z.object({}),
      },
    },
  });

  return result.toUIMessageStreamResponse();
}
```

### 클라이언트 페이지

클라이언트 페이지는 `useChat` 훅을 사용해 실시간 메시지 스트리밍이 되는 챗봇 앱을 만들고,
툴 호출은 타입이 지정된 툴 파트로 채팅 UI에 표시돼.
반드시 메시지를 렌더링할 때 `parts` 속성을 사용해줘.

세 가지 포인트만 짚자면:

1. [`onToolCall`](/docs/reference/ai-sdk-ui/use-chat#on-tool-call) 콜백은 클라이언트에서 자동 실행되어야 하는 툴을 처리하는 데 사용돼.
   이 예시에서 `getLocation`은 랜덤 도시를 반환하는 클라이언트 툴이야.
   결과를 제공하려면 `addToolResult`를 호출해(잠재적 데드락 방지를 위해 `await` 없이).

   <Note>
     `onToolCall` 핸들러에서 먼저 `if (toolCall.dynamic)` 체크를 해줘.
     이 체크가 없으면 `addToolResult`에서 `toolCall.toolName`을 사용할 때
     TypeScript가 `Type 'string' is not assignable to type '"toolName1" | "toolName2"'`
     같은 에러를 낼 수 있어.
   </Note>

2. [`sendAutomaticallyWhen`](/docs/reference/ai-sdk-ui/use-chat#send-automatically-when) 옵션과 `lastAssistantMessageIsCompleteWithToolCalls` 헬퍼를 함께 쓰면, 모든 툴 결과가 준비됐을 때 자동 제출돼.

3. 어시스턴트 메시지의 `parts` 배열은 `tool-askForConfirmation` 같은 타입 이름을 가진 툴 파트를 포함해.
   클라이언트 툴 `askForConfirmation`는 UI에 표시되고,
   사용자에게 확인을 요청한 뒤, 승인/거절 결과를 보여줘.
   이 결과는 타입 안전을 위해 `tool` 파라미터와 함께 `addToolResult`로 채팅에 추가해.

```tsx filename='app/page.tsx' highlight="2,6,10,14-20"
'use client';

import { useChat } from '@ai-sdk/react';
import {
  DefaultChatTransport,
  lastAssistantMessageIsCompleteWithToolCalls,
} from 'ai';
import { useState } from 'react';

export default function Chat() {
  const { messages, sendMessage, addToolResult } = useChat({
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),

    sendAutomaticallyWhen: lastAssistantMessageIsCompleteWithToolCalls,

    // run client-side tools that are automatically executed:
    async onToolCall({ toolCall }) {
      // Check if it's a dynamic tool first for proper type narrowing
      if (toolCall.dynamic) {
        return;
      }

      if (toolCall.toolName === 'getLocation') {
        const cities = ['New York', 'Los Angeles', 'Chicago', 'San Francisco'];

        // No await - avoids potential deadlocks
        addToolResult({
          tool: 'getLocation',
          toolCallId: toolCall.toolCallId,
          output: cities[Math.floor(Math.random() * cities.length)],
        });
      }
    },
  });
  const [input, setInput] = useState('');

  return (
    <>
      {messages?.map(message => (
        <div key={message.id}>
          <strong>{`${message.role}: `}</strong>
          {message.parts.map(part => {
            switch (part.type) {
              // render text parts as simple text:
              case 'text':
                return part.text;

              // for tool parts, use the typed tool part names:
              case 'tool-askForConfirmation': {
                const callId = part.toolCallId;

                switch (part.state) {
                  case 'input-streaming':
                    return (
                      <div key={callId}>Loading confirmation request...</div>
                    );
                  case 'input-available':
                    return (
                      <div key={callId}>
                        {part.input.message}
                        <div>
                          <button
                            onClick={() =>
                              addToolResult({
                                tool: 'askForConfirmation',
                                toolCallId: callId,
                                output: 'Yes, confirmed.',
                              })
                            }
                          >
                            Yes
                          </button>
                          <button
                            onClick={() =>
                              addToolResult({
                                tool: 'askForConfirmation',
                                toolCallId: callId,
                                output: 'No, denied',
                              })
                            }
                          >
                            No
                          </button>
                        </div>
                      </div>
                    );
                  case 'output-available':
                    return (
                      <div key={callId}>
                        Location access allowed: {part.output}
                      </div>
                    );
                  case 'output-error':
                    return <div key={callId}>Error: {part.errorText}</div>;
                }
                break;
              }

              case 'tool-getLocation': {
                const callId = part.toolCallId;

                switch (part.state) {
                  case 'input-streaming':
                    return (
                      <div key={callId}>Preparing location request...</div>
                    );
                  case 'input-available':
                    return <div key={callId}>Getting location...</div>;
                  case 'output-available':
                    return <div key={callId}>Location: {part.output}</div>;
                  case 'output-error':
                    return (
                      <div key={callId}>
                        Error getting location: {part.errorText}
                      </div>
                    );
                }
                break;
              }

              case 'tool-getWeatherInformation': {
                const callId = part.toolCallId;

                switch (part.state) {
                  // example of pre-rendering streaming tool inputs:
                  case 'input-streaming':
                    return (
                      <pre key={callId}>{JSON.stringify(part, null, 2)}</pre>
                    );
                  case 'input-available':
                    return (
                      <div key={callId}>
                        Getting weather information for {part.input.city}...
                      </div>
                    );
                  case 'output-available':
                    return (
                      <div key={callId}>
                        Weather in {part.input.city}: {part.output}
                      </div>
                    );
                  case 'output-error':
                    return (
                      <div key={callId}>
                        Error getting weather for {part.input.city}:{' '}
                        {part.errorText}
                      </div>
                    );
                }
                break;
              }
            }
          })}
          <br />
        </div>
      ))}

      <form
        onSubmit={e => {
          e.preventDefault();
          if (input.trim()) {
            sendMessage({ text: input });
            setInput('');
          }
        }}
      >
        <input value={input} onChange={e => setInput(e.target.value)} />
      </form>
    </>
  );
}
```

## 동적(Dynamic) 툴

컴파일 타임에 타입을 알 수 없는 동적 툴을 사용할 때, UI 파트는 특정 툴 타입 대신 일반적인 `dynamic-tool` 타입을 사용해:

```tsx filename='app/page.tsx'
{
  message.parts.map((part, index) => {
    switch (part.type) {
      // Static tools with specific (`tool-${toolName}`) types
      case 'tool-getWeatherInformation':
        return <WeatherDisplay part={part} />;

      // Dynamic tools use generic `dynamic-tool` type
      case 'dynamic-tool':
        return (
          <div key={index}>
            <h4>Tool: {part.toolName}</h4>
            {part.state === 'input-streaming' && (
              <pre>{JSON.stringify(part.input, null, 2)}</pre>
            )}
            {part.state === 'output-available' && (
              <pre>{JSON.stringify(part.output, null, 2)}</pre>
            )}
            {part.state === 'output-error' && (
              <div>Error: {part.errorText}</div>
            )}
          </div>
        );
    }
  });
}
```

동적 툴은 다음과 같이 통합할 때 유용해:

- 스키마가 없는 MCP(Model Context Protocol) 툴
- 런타임에 로드되는 사용자 정의 함수
- 외부 툴 제공자

## 툴 콜 스트리밍

AI SDK 5.0에서는 기본적으로 툴 콜 스트리밍이 활성화돼 있어서, 생성 중인 툴 콜을 스트리밍할 수 있어. 이렇게 하면 툴 입력이 실시간으로 생성될 때 즉시 보여줄 수 있어 더 좋은 UX를 제공해.

```tsx filename='app/api/chat/route.ts'
export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages: convertToModelMessages(messages),
    // toolCallStreaming is enabled by default in v5
    // ...
  });

  return result.toUIMessageStreamResponse();
}
```

툴 콜 스트리밍이 활성화되어 있으면, 부분(Partial) 툴 콜이 데이터 스트림의 일부로 전송돼.
이 데이터는 `useChat` 훅을 통해 접근할 수 있고,
어시스턴트 메시지의 타입 지정된 툴 파트에도 부분 툴 콜이 포함돼.
`state` 속성을 사용해 알맞은 UI를 렌더링하면 돼.

```tsx filename='app/page.tsx' highlight="9,10"
export default function Chat() {
  // ...
  return (
    <>
      {messages?.map(message => (
        <div key={message.id}>
          {message.parts.map(part => {
            switch (part.type) {
              case 'tool-askForConfirmation':
              case 'tool-getLocation':
              case 'tool-getWeatherInformation':
                switch (part.state) {
                  case 'input-streaming':
                    return <pre>{JSON.stringify(part.input, null, 2)}</pre>;
                  case 'input-available':
                    return <pre>{JSON.stringify(part.input, null, 2)}</pre>;
                  case 'output-available':
                    return <pre>{JSON.stringify(part.output, null, 2)}</pre>;
                  case 'output-error':
                    return <div>Error: {part.errorText}</div>;
                }
            }
          })}
        </div>
      ))}
    </>
  );
}
```

## Step 시작 파트

멀티 스텝 툴 콜을 사용할 때, AI SDK는 어시스턴트 메시지에 step 시작 파트를 추가해.
툴 콜 사이의 경계를 표시하고 싶다면, `step-start` 파트를 다음처럼 사용해:

```tsx filename='app/page.tsx'
// ...
// where you render the message parts:
message.parts.map((part, index) => {
  switch (part.type) {
    case 'step-start':
      // show step boundaries as horizontal lines:
      return index > 0 ? (
        <div key={index} className="text-gray-500">
          <hr className="my-2 border-gray-300" />
        </div>
      ) : null;
    case 'text':
    // ...
    case 'tool-askForConfirmation':
    case 'tool-getLocation':
    case 'tool-getWeatherInformation':
    // ...
  }
});
// ...
```

## 서버 사이드 멀티 스텝 콜

`streamText`로 서버 사이드에서도 멀티 스텝 콜을 사용할 수 있어.
이는 호출된 모든 툴이 서버에서 `execute` 함수를 가지고 있을 때 동작해.

```tsx filename='app/api/chat/route.ts' highlight="15-21,24"
import { openai } from '@ai-sdk/openai';
import { convertToModelMessages, streamText, UIMessage, stepCountIs } from 'ai';
import { z } from 'zod';

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages: convertToModelMessages(messages),
    tools: {
      getWeatherInformation: {
        description: 'show the weather in a given city to the user',
        inputSchema: z.object({ city: z.string() }),
        // tool has execute function:
        execute: async ({}: { city: string }) => {
          const weatherOptions = ['sunny', 'cloudy', 'rainy', 'snowy', 'windy'];
          return weatherOptions[
            Math.floor(Math.random() * weatherOptions.length)
          ];
        },
      },
    },
    stopWhen: stepCountIs(5),
  });

  return result.toUIMessageStreamResponse();
}
```

## 에러 처리

LLM은 툴을 호출할 때 오류를 낼 수 있어.
기본적으로 보안상의 이유로 이러한 오류는 마스킹되어 UI에는 "An error occurred"로 표시돼.

오류를 노출하려면, `toUIMessageResponse` 호출 시 `onError` 함수를 사용할 수 있어.

```tsx
export function errorHandler(error: unknown) {
  if (error == null) {
    return 'unknown error';
  }

  if (typeof error === 'string') {
    return error;
  }

  if (error instanceof Error) {
    return error.message;
  }

  return JSON.stringify(error);
}
```

```tsx
const result = streamText({
  // ...
});

return result.toUIMessageStreamResponse({
  onError: errorHandler,
});
```

`createUIMessageResponse`를 사용하는 경우라면, `toUIMessageResponse` 호출 시 `onError`를 다음처럼 지정할 수 있어:

```tsx
const response = createUIMessageResponse({
  // ...
  async execute(dataStream) {
    // ...
  },
  onError: error => `Custom error: ${error.message}`,
});
```
