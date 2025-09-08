# Chatbot (챗봇)

`useChat` 훅은 챗봇 앱을 위한 대화형 UI를 쉽게 만들 수 있게 해줘. AI 제공자에서 오는 채팅 메시지를 스트리밍하고, 채팅 상태를 관리하며, 새 메시지가 도착할 때 자동으로 UI를 업데이트해.

요약하면, `useChat` 훅은 다음 기능을 제공해:

- 메시지 스트리밍: AI 제공자의 모든 메시지를 실시간으로 UI에 스트리밍
- 상태 관리: 입력, 메시지, 상태, 에러 등 상태를 훅이 알아서 관리
- 쉬운 통합: 최소한의 작업으로 어떤 디자인/레이아웃에도 쉽게 통합

이 가이드에선 `useChat` 훅을 사용해 실시간 메시지 스트리밍이 되는 챗봇을 만드는 방법을 배울 거야.
툴 사용이 필요한 경우 [챗봇 + 툴 가이드](/docs/ai-sdk-ui/chatbot-with-tool-calling)도 확인해 봐.
먼저 아래 예시부터 시작하자.

## Example

```tsx filename='app/page.tsx'
'use client';

import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';
import { useState } from 'react';

export default function Page() {
  const { messages, sendMessage, status } = useChat({
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),
  });
  const [input, setInput] = useState('');

  return (
    <>
      {messages.map(message => (
        <div key={message.id}>
          {message.role === 'user' ? 'User: ' : 'AI: '}
          {message.parts.map((part, index) =>
            part.type === 'text' ? <span key={index}>{part.text}</span> : null,
          )}
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
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          disabled={status !== 'ready'}
          placeholder="Say something..."
        />
        <button type="submit" disabled={status !== 'ready'}>
          Submit
        </button>
      </form>
    </>
  );
}
```

```ts filename='app/api/chat/route.ts'
import { openai } from '@ai-sdk/openai';
import { convertToModelMessages, streamText, UIMessage } from 'ai';

// Allow streaming responses up to 30 seconds
export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4.1'),
    system: 'You are a helpful assistant.',
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse();
}
```

<Note>
  UI 메시지에는 메시지 파트를 담는 새로운 `parts` 프로퍼티가 있어.
  `content` 대신 `parts`를 써서 렌더링하는 걸 추천해. `parts`는 텍스트, 툴 호출,
  툴 결과 등 다양한 메시지 타입을 지원해서 더 유연하고 복잡한 챗 UI를 만들 수 있어.
</Note>

`Page` 컴포넌트에서, 사용자가 `sendMessage`로 메시지를 보낼 때마다 `useChat` 훅이 AI 제공자 엔드포인트로 요청을 보내.
응답 메시지는 실시간으로 스트리밍되어 UI에 표시돼.

이렇게 하면 전체 응답을 다 기다리지 않고, 준비되는 대로 바로바로 볼 수 있는 경험을 제공해.

## Customized UI

`useChat`은 코드로 메시지 상태를 직접 관리하고, 상태 표시를 하거나, 사용자 인터랙션 없이 메시지를 업데이트하는 것도 가능해.

### Status

`useChat`이 반환하는 `status`는 다음 값 중 하나야:

- `submitted`: 메시지가 API로 전송되었고, 스트림 시작을 기다리는 중
- `streaming`: API로부터 응답이 스트리밍되는 중
- `ready`: 전체 응답 수신 및 처리 완료; 새 사용자 메시지 전송 가능
- `error`: API 요청 중 에러가 발생해 완료하지 못함

`status`는 예를 들어 다음에 활용할 수 있어:

- 처리 중 로딩 스피너 표시
- 현재 메시지 중단을 위한 "Stop" 버튼 표시
- 제출 버튼 비활성화

```tsx filename='app/page.tsx' highlight="6,22-29,36"
'use client';

import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';
import { useState } from 'react';

export default function Page() {
  const { messages, sendMessage, status, stop } = useChat({
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),
  });
  const [input, setInput] = useState('');

  return (
    <>
      {messages.map(message => (
        <div key={message.id}>
          {message.role === 'user' ? 'User: ' : 'AI: '}
          {message.parts.map((part, index) =>
            part.type === 'text' ? <span key={index}>{part.text}</span> : null,
          )}
        </div>
      ))}

      {(status === 'submitted' || status === 'streaming') && (
        <div>
          {status === 'submitted' && <Spinner />}
          <button type="button" onClick={() => stop()}>
            Stop
          </button>
        </div>
      )}

      <form
        onSubmit={e => {
          e.preventDefault();
          if (input.trim()) {
            sendMessage({ text: input });
            setInput('');
          }
        }}
      >
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          disabled={status !== 'ready'}
          placeholder="Say something..."
        />
        <button type="submit" disabled={status !== 'ready'}>
          Submit
        </button>
      </form>
    </>
  );
}
```

### Error State

`error` 상태는 fetch 요청 중 던져진 에러 객체를 반영해. 에러 메시지 표시, 제출 버튼 비활성화, 재시도 버튼 표시 등에 쓸 수 있어.

<Note>
  사용자에게는 "Something went wrong." 같은 일반적인 에러 메시지를 보여주는 걸 추천해. 서버 정보 누출 방지를 위해 좋아.
</Note>

```tsx file="app/page.tsx" highlight="6,20-27,33"
'use client';

import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';
import { useState } from 'react';

export default function Chat() {
  const { messages, sendMessage, error, reload } = useChat({
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),
  });
  const [input, setInput] = useState('');

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.role}:{' '}
          {m.parts.map((part, index) =>
            part.type === 'text' ? <span key={index}>{part.text}</span> : null,
          )}
        </div>
      ))}

      {error && (
        <>
          <div>An error occurred.</div>
          <button type="button" onClick={() => reload()}>
            Retry
          </button>
        </>
      )}

      <form
        onSubmit={e => {
          e.preventDefault();
          if (input.trim()) {
            sendMessage({ text: input });
            setInput('');
          }
        }}
      >
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          disabled={error != null}
        />
      </form>
    </div>
  );
}
```

자세한 내용은 [error handling](/docs/ai-sdk-ui/error-handling) 가이드를 참고해.

### Modify messages

가끔 기존 메시지를 직접 수정하고 싶을 때가 있어. 예를 들어 각 메시지에 삭제 버튼을 넣어서 히스토리에서 제거할 수 있게 할 수 있지.

`setMessages` 함수로 이런 작업을 할 수 있어:

```tsx
const { messages, setMessages } = useChat()

const handleDelete = (id) => {
  setMessages(messages.filter(message => message.id !== id))
}

return <>
  {messages.map(message => (
    <div key={message.id}>
      {message.role === 'user' ? 'User: ' : 'AI: '}
      {message.parts.map((part, index) => (
        part.type === 'text' ? (
          <span key={index}>{part.text}</span>
        ) : null
      ))}
      <button onClick={() => handleDelete(message.id)}>Delete</button>
    </div>
  ))}
  ...
```

`messages`와 `setMessages`를 React의 `state`/`setState` 쌍으로 생각하면 돼.

### Cancellation and regeneration

AI 제공자로부터 응답이 스트리밍되는 도중에 메시지를 중단하는 것도 흔한 요구야. `useChat`이 반환하는 `stop` 함수를 호출하면 돼.

```tsx
const { stop, status } = useChat()

return <>
  <button onClick={stop} disabled={!(status === 'streaming' || status === 'submitted')}>Stop</button>
  ...
```

사용자가 "Stop"을 클릭하면 fetch 요청이 중단돼. 불필요한 리소스 소모를 막고 UX를 개선해.

비슷하게, `useChat`의 `regenerate`로 마지막 메시지를 다시 생성하도록 요청할 수도 있어:

```tsx
const { regenerate, status } = useChat();

return (
  <>
    <button
      onClick={regenerate}
      disabled={!(status === 'ready' || status === 'error')}
    >
      Regenerate
    </button>
    ...
  </>
);
```

사용자가 "Regenerate"를 클릭하면, 마지막 메시지를 다시 생성하고 기존 메시지를 교체해.

### Throttling UI Updates

<Note>이 기능은 현재 React에서만 지원돼.</Note>

기본적으로 `useChat`은 새 청크를 받을 때마다 렌더를 트리거해. `experimental_throttle` 옵션으로 UI 업데이트를 쓰로틀링할 수 있어.

```tsx filename="page.tsx" highlight="2-3"
const { messages, ... } = useChat({
  // Throttle the messages and data updates to 50ms:
  experimental_throttle: 50
})
```

## Event Callbacks

`useChat`은 챗봇 라이프사이클의 다양한 단계에서 처리할 수 있는 선택적 이벤트 콜백을 제공해:

- `onFinish`: 어시스턴트 응답이 완료되었을 때 호출. 응답 메시지, 전체 메시지, abort/disconnect/error 플래그 포함
- `onError`: fetch 요청 중 에러 발생 시 호출
- `onData`: 데이터 파트를 받을 때마다 호출

이 콜백들로 로깅, 분석, 커스텀 UI 업데이트 같은 추가 작업을 트리거할 수 있어.

```tsx
import { UIMessage } from 'ai';

const {
  /* ... */
} = useChat({
  onFinish: ({ message, messages, isAbort, isDisconnect, isError }) => {
    // 예: 다른 UI 상태 업데이트
  },
  onError: error => {
    console.error('An error occurred:', error);
  },
  onData: data => {
    console.log('Received data part from server:', data);
  },
});
```

`onData` 콜백에서 에러를 throw하면 처리를 중단할 수 있어. 이 경우 `onError`가 호출되고, 해당 메시지는 챗 UI에 추가되지 않아. 예기치 못한 응답을 처리할 때 유용해.

## Request Configuration

### 커스텀 headers, body, credentials

기본적으로 `useChat`은 `/api/chat` 엔드포인트로 메시지 리스트를 요청 바디에 담아 HTTP POST를 보냄. 요청 커스터마이즈 방법은 두 가지야:

#### 훅 레벨 설정(모든 요청에 적용)

훅이 보내는 모든 요청에 적용될 트랜스포트 옵션을 설정할 수 있어:

```tsx
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

const { messages, sendMessage } = useChat({
  transport: new DefaultChatTransport({
    api: '/api/custom-chat',
    headers: {
      Authorization: 'your_token',
    },
    body: {
      user_id: '123',
    },
    credentials: 'same-origin',
  }),
});
```

#### 동적 훅 레벨 설정

값을 반환하는 함수를 제공할 수도 있어. 토큰 갱신처럼 런타임 조건에 의존하는 설정에 유용해:

```tsx
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

const { messages, sendMessage } = useChat({
  transport: new DefaultChatTransport({
    api: '/api/custom-chat',
    headers: () => ({
      Authorization: `Bearer ${getAuthToken()}`,
      'X-User-ID': getCurrentUserId(),
    }),
    body: () => ({
      sessionId: getCurrentSessionId(),
      preferences: getUserPreferences(),
    }),
    credentials: () => 'include',
  }),
});
```

<Note>
  시간이 지나며 변하는 컴포넌트 상태는 `useRef`에 저장한 뒤 `ref.current`를 설정 함수에서 참조하거나,
  더 신뢰성 있게는 요청 레벨 옵션(다음 섹션)을 쓰는 걸 추천해.
</Note>

#### 요청 레벨 설정(추천)

<Note>
  추천: 요청 레벨 옵션은 유연성과 제어력이 좋아. 훅 레벨 옵션 위에 우선 적용되고, 각 요청을 개별적으로 커스터마이즈할 수 있어.
</Note>

```tsx
// sendMessage의 두 번째 파라미터로 옵션 전달
sendMessage(
  { text: input },
  {
    headers: {
      Authorization: 'Bearer token123',
      'X-Custom-Header': 'custom-value',
    },
    body: {
      temperature: 0.7,
      max_tokens: 100,
      user_id: '123',
    },
    metadata: {
      userId: 'user123',
      sessionId: 'session456',
    },
  },
);
```

요청 레벨 옵션은 훅 레벨 옵션과 머지되며, 요청 레벨이 우선이야. 서버에선 이 추가 정보를 사용해 요청을 처리하면 돼.

### 요청별 커스텀 body 필드 설정

`sendMessage`의 두 번째 파라미터로 요청마다 커스텀 `body` 필드를 보낼 수 있어. 메시지 리스트에 포함되지 않는 추가 정보를 백엔드로 보내고 싶을 때 유용해.

```tsx filename="app/page.tsx" highlight="20-25"
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export default function Chat() {
  const { messages, sendMessage } = useChat();
  const [input, setInput] = useState('');

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.role}:{' '}
          {m.parts.map((part, index) =>
            part.type === 'text' ? <span key={index}>{part.text}</span> : null,
          )}
        </div>
      ))}

      <form
        onSubmit={event => {
          event.preventDefault();
          if (input.trim()) {
            sendMessage(
              { text: input },
              {
                body: {
                  customKey: 'customValue',
                },
              },
            );
            setInput('');
          }
        }}
      >
        <input value={input} onChange={e => setInput(e.target.value)} />
      </form>
    </div>
  );
}
```

서버에서는 요청 바디 구조 분해로 이 커스텀 필드를 받을 수 있어:

```ts filename="app/api/chat/route.ts" highlight="3,4"
export async function POST(req: Request) {
  // Extract additional information ("customKey") from the body of the request:
  const { messages, customKey }: { messages: UIMessage[]; customKey: string } =
    await req.json();
  //...
}
```

## Message Metadata

타임스탬프, 모델 정보, 토큰 사용량 같은 추적 정보를 메시지에 메타데이터로 붙일 수 있어.

```ts
// Server: Send metadata about the message
return result.toUIMessageStreamResponse({
  messageMetadata: ({ part }) => {
    if (part.type === 'start') {
      return {
        createdAt: Date.now(),
        model: 'gpt-4o',
      };
    }

    if (part.type === 'finish') {
      return {
        totalTokens: part.totalUsage.totalTokens,
      };
    }
  },
});
```

```tsx
// Client: Access metadata via message.metadata
{
  messages.map(message => (
    <div key={message.id}>
      {message.role}:{' '}
      {message.metadata?.createdAt &&
        new Date(message.metadata.createdAt).toLocaleTimeString()}
      {/* Render message content */}
      {message.parts.map((part, index) =>
        part.type === 'text' ? <span key={index}>{part.text}</span> : null,
      )}
      {/* Show token count if available */}
      {message.metadata?.totalTokens && (
        <span>{message.metadata.totalTokens} tokens</span>
      )}
    </div>
  ));
}
```

타입 안전성과 고급 사례 전체 예시는 [Message Metadata 문서](/docs/ai-sdk-ui/message-metadata)에서 볼 수 있어.

## Transport Configuration

`transport` 옵션으로 메시지가 API에 전송되는 방식을 커스터마이즈할 수 있어:

```tsx filename="app/page.tsx"
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

export default function Chat() {
  const { messages, sendMessage } = useChat({
    id: 'my-chat',
    transport: new DefaultChatTransport({
      prepareSendMessagesRequest: ({ id, messages }) => {
        return {
          body: {
            id,
            message: messages[messages.length - 1],
          },
        };
      },
    }),
  });

  // ... rest of your component
}
```

서버의 API 라우트는 커스텀 요청 포맷을 다음처럼 받게 돼:

```ts filename="app/api/chat/route.ts"
export async function POST(req: Request) {
  const { id, message } = await req.json();

  // Load existing messages and add the new one
  const messages = await loadMessages(id);
  messages.push(message);

  const result = streamText({
    model: openai('gpt-4.1'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse();
}
```

### Advanced: Trigger-based routing

재생성 같은 복잡한 시나리오엔 트리거 기반 라우팅을 사용할 수 있어:

```tsx filename="app/page.tsx"
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

export default function Chat() {
  const { messages, sendMessage, regenerate } = useChat({
    id: 'my-chat',
    transport: new DefaultChatTransport({
      prepareSendMessagesRequest: ({ id, messages, trigger, messageId }) => {
        if (trigger === 'submit-user-message') {
          return {
            body: {
              trigger: 'submit-user-message',
              id,
              message: messages[messages.length - 1],
              messageId,
            },
          };
        } else if (trigger === 'regenerate-assistant-message') {
          return {
            body: {
              trigger: 'regenerate-assistant-message',
              id,
              messageId,
            },
          };
        }
        throw new Error(`Unsupported trigger: ${trigger}`);
      },
    }),
  });

  // ... rest of your component
}
```

서버의 API 라우트는 트리거 종류에 따라 다르게 처리하면 돼:

```ts filename="app/api/chat/route.ts"
export async function POST(req: Request) {
  const { trigger, id, message, messageId } = await req.json();

  const chat = await readChat(id);
  let messages = chat.messages;

  if (trigger === 'submit-user-message') {
    // Handle new user message
    messages = [...messages, message];
  } else if (trigger === 'regenerate-assistant-message') {
    // Handle message regeneration - remove messages after messageId
    const messageIndex = messages.findIndex(m => m.id === messageId);
    if (messageIndex !== -1) {
      messages = messages.slice(0, messageIndex);
    }
  }

  const result = streamText({
    model: openai('gpt-4.1'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse();
}
```

더 자세한 트랜스포트 커스터마이징은 [Transport API 문서](/docs/ai-sdk-ui/transport)를 참고해.

## Controlling the response stream

`streamText`로 에러 메시지와 사용량 정보를 클라이언트에 어떻게 보낼지 제어할 수 있어.

### Error Messages

기본적으로 보안상의 이유로 에러 메시지는 마스킹돼. 기본 메시지는 "An error occurred."야.
`getErrorMessage` 함수를 제공해 에러 메시지를 전달하거나 커스텀 메시지를 보낼 수 있어:

```ts filename="app/api/chat/route.ts" highlight="13-27"
import { openai } from '@ai-sdk/openai';
import { convertToModelMessages, streamText, UIMessage } from 'ai';

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4.1'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse({
    onError: error => {
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
    },
  });
}
```

### Usage Information

[message metadata](/docs/ai-sdk-ui/message-metadata)로 토큰/리소스 사용량을 추적해:

1. 사용량 필드를 가진 커스텀 메타데이터 타입 정의(선택)
2. 응답에서 `messageMetadata`로 사용량 데이터 첨부
3. UI 컴포넌트에서 사용량 지표 표시

사용량 데이터는 메시지 메타데이터로 붙고, 모델이 응답 생성을 완료하면 이용 가능해져.

```ts
import { openai } from '@ai-sdk/openai';
import {
  convertToModelMessages,
  streamText,
  UIMessage,
  type LanguageModelUsage,
} from 'ai';

// Create a new metadata type (optional for type-safety)
type MyMetadata = {
  totalUsage: LanguageModelUsage;
};

// Create a new custom message type with your own metadata
export type MyUIMessage = UIMessage<MyMetadata>;

export async function POST(req: Request) {
  const { messages }: { messages: MyUIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse({
    originalMessages: messages,
    messageMetadata: ({ part }) => {
      // Send total usage when generation is finished
      if (part.type === 'finish') {
        return { totalUsage: part.totalUsage };
      }
    },
  });
}
```

클라이언트에선 메시지 단위 메타데이터로 접근할 수 있어.

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import type { MyUIMessage } from './api/chat/route';
import { DefaultChatTransport } from 'ai';

export default function Chat() {
  // 서버에서 정의한 커스텀 메시지 타입 사용(선택)
  const { messages } = useChat<MyUIMessage>({
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),
  });

  return (
    <div className="flex flex-col w-full max-w-md py-24 mx-auto stretch">
      {messages.map(m => (
        <div key={m.id} className="whitespace-pre-wrap">
          {m.role === 'user' ? 'User: ' : 'AI: '}
          {m.parts.map(part => {
            if (part.type === 'text') {
              return part.text;
            }
          })}
          {/* Render usage via metadata */}
          {m.metadata?.totalUsage && (
            <div>Total usage: {m.metadata?.totalUsage.totalTokens} tokens</div>
          )}
        </div>
      ))}
    </div>
  );
}
```

`useChat`의 `onFinish` 콜백에서도 메타데이터에 접근할 수 있어:

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import type { MyUIMessage } from './api/chat/route';
import { DefaultChatTransport } from 'ai';

export default function Chat() {
  // Use custom message type defined on the server (optional for type-safety)
  const { messages } = useChat<MyUIMessage>({
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),
    onFinish: ({ message }) => {
      // Access message metadata via onFinish callback
      console.log(message.metadata?.totalUsage);
    },
  });
}
```

### Text Streams

`useChat`은 `streamProtocol` 옵션을 `text`로 설정해 단순 텍스트 스트림도 처리할 수 있어:

```tsx filename="app/page.tsx" highlight="7"
'use client';

import { useChat } from '@ai-sdk/react';
import { TextStreamChatTransport } from 'ai';

export default function Chat() {
  const { messages } = useChat({
    transport: new TextStreamChatTransport({
      api: '/api/chat',
    }),
  });

  return <>...</>;
}
```

이 설정은 일반 텍스트를 스트리밍하는 다른 백엔드 서버와도 잘 작동해.
자세한 내용은 [stream protocol 가이드](/docs/ai-sdk-ui/stream-protocol)를 참고해.

<Note>
  `TextStreamChatTransport`를 사용할 때는 툴 호출, 사용량 정보, 종료 사유가 제공되지 않아.
</Note>

## Reasoning

DeepSeek `deepseek-reasoner`나 Anthropic `claude-3-7-sonnet-20250219` 같은 일부 모델은 reasoning 토큰을 지원해.
이 토큰들은 보통 메시지 콘텐츠 전에 전송돼. `sendReasoning` 옵션으로 클라이언트에 전달할 수 있어:

```ts filename="app/api/chat/route.ts" highlight="13"
import { deepseek } from '@ai-sdk/deepseek';
import { convertToModelMessages, streamText, UIMessage } from 'ai';

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: deepseek('deepseek-reasoner'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse({
    sendReasoning: true,
  });
}
```

클라이언트에서는 메시지 객체의 reasoning 파트에 접근하면 돼. reasoning 파트는 추론 내용을 담는 `text` 프로퍼티를 가져.

```tsx filename="app/page.tsx"
messages.map(message => (
  <div key={message.id}>
    {message.role === 'user' ? 'User: ' : 'AI: '}
    {message.parts.map((part, index) => {
      // text parts:
      if (part.type === 'text') {
        return <div key={index}>{part.text}</div>;
      }

      // reasoning parts:
      if (part.type === 'reasoning') {
        return <pre key={index}>{part.text}</pre>;
      }
    })}
  </div>
));
```

## Sources

[Perplexity](/providers/ai-sdk-providers/perplexity#sources)나
[Google Generative AI](/providers/ai-sdk-providers/google-generative-ai#sources) 같은 일부 제공자는 응답에 소스를 포함해.

현재 소스는 응답을 근거지우는 웹페이지로 제한돼. `sendSources` 옵션으로 클라이언트에 전달할 수 있어:

```ts filename="app/api/chat/route.ts" highlight="13"
import { perplexity } from '@ai-sdk/perplexity';
import { convertToModelMessages, streamText, UIMessage } from 'ai';

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: perplexity('sonar-pro'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse({
    sendSources: true,
  });
}
```

클라이언트에서는 메시지 객체의 소스 파트에 접근하면 돼. 소스에는 두 가지 타입이 있어: 웹페이지용 `source-url`, 문서용 `source-document`.
아래는 두 타입을 모두 렌더링하는 예시야:

```tsx filename="app/page.tsx"
messages.map(message => (
  <div key={message.id}>
    {message.role === 'user' ? 'User: ' : 'AI: '}

    {/* Render URL sources */}
    {message.parts
      .filter(part => part.type === 'source-url')
      .map(part => (
        <span key={`source-${part.id}`}>
          [
          <a href={part.url} target="_blank">
            {part.title ?? new URL(part.url).hostname}
          </a>
          ]
        </span>
      ))}

    {/* Render document sources */}
    {message.parts
      .filter(part => part.type === 'source-document')
      .map(part => (
        <span key={`source-${part.id}`}>
          [<span>{part.title ?? `Document ${part.id}`}</span>]
        </span>
      ))}
  </div>
));
```

## Image Generation

Google `gemini-2.0-flash-exp` 같은 일부 모델은 이미지 생성을 지원해. 생성된 이미지는 클라이언트에 파일로 전달돼. 클라이언트에선 메시지 객체의 파일 파트에 접근해서 이미지를 렌더링하면 돼.

```tsx filename="app/page.tsx"
messages.map(message => (
  <div key={message.id}>
    {message.role === 'user' ? 'User: ' : 'AI: '}
    {message.parts.map((part, index) => {
      if (part.type === 'text') {
        return <div key={index}>{part.text}</div>;
      } else if (part.type === 'file' && part.mediaType.startsWith('image/')) {
        return <img key={index} src={part.url} alt="Generated image" />;
      }
    })}
  </div>
));
```

## Attachments

`useChat` 훅은 메시지와 함께 파일 첨부를 보내고, 클라이언트에서 렌더링하는 걸 지원해. 이미지/파일/기타 미디어를 AI 제공자에 보내는 앱을 만들 때 유용해.

파일을 보내는 방법은 두 가지야: 파일 입력에서 얻은 `FileList`를 쓰거나, 파일 객체 배열을 직접 전달.

### FileList

`FileList`를 사용하면 파일 입력 요소로 여러 파일을 메시지에 첨부해서 보낼 수 있어. `useChat` 훅이 자동으로 data URL로 변환해 AI 제공자에 전송해 줘.

<Note>
  현재 자동 변환은 `image/*`와 `text/*` 콘텐츠 타입만 지원해.
  다른 타입은 멀티모달 콘텐츠 파트로 직접 처리해야 해.
</Note>

```tsx filename="app/page.tsx"
'use client';

import { useChat } from '@ai-sdk/react';
import { useRef, useState } from 'react';

export default function Page() {
  const { messages, sendMessage, status } = useChat();

  const [input, setInput] = useState('');
  const [files, setFiles] = useState<FileList | undefined>(undefined);
  const fileInputRef = useRef<HTMLInputElement>(null);

  return (
    <div>
      <div>
        {messages.map(message => (
          <div key={message.id}>
            <div>{`${message.role}: `}</div>

            <div>
              {message.parts.map((part, index) => {
                if (part.type === 'text') {
                  return <span key={index}>{part.text}</span>;
                }

                if (
                  part.type === 'file' &&
                  part.mediaType?.startsWith('image/')
                ) {
                  return <img key={index} src={part.url} alt={part.filename} />;
                }

                return null;
              })}
            </div>
          </div>
        ))}
      </div>

      <form
        onSubmit={event => {
          event.preventDefault();
          if (input.trim()) {
            sendMessage({
              text: input,
              files,
            });
            setInput('');
            setFiles(undefined);

            if (fileInputRef.current) {
              fileInputRef.current.value = '';
            }
          }
        }}
      >
        <input
          type="file"
          onChange={event => {
            if (event.target.files) {
              setFiles(event.target.files);
            }
          }}
          multiple
          ref={fileInputRef}
        />
        <input
          value={input}
          placeholder="Send message..."
          onChange={e => setInput(e.target.value)}
          disabled={status !== 'ready'}
        />
      </form>
    </div>
  );
}
```

### File Objects

파일 객체를 메시지와 함께 보낼 수도 있어. 미리 업로드된 파일이나 data URL을 보낼 때 유용해.

```tsx filename="app/page.tsx"
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';
import { FileUIPart } from 'ai';

export default function Page() {
  const { messages, sendMessage, status } = useChat();

  const [input, setInput] = useState('');
  const [files] = useState<FileUIPart[]>([
    {
      type: 'file',
      filename: 'earth.png',
      mediaType: 'image/png',
      url: 'https://example.com/earth.png',
    },
    {
      type: 'file',
      filename: 'moon.png',
      mediaType: 'image/png',
      url: 'data:image/png;base64,iVBORw0KGgo...',
    },
  ]);

  return (
    <div>
      <div>
        {messages.map(message => (
          <div key={message.id}>
            <div>{`${message.role}: `}</div>

            <div>
              {message.parts.map((part, index) => {
                if (part.type === 'text') {
                  return <span key={index}>{part.text}</span>;
                }

                if (
                  part.type === 'file' &&
                  part.mediaType?.startsWith('image/')
                ) {
                  return <img key={index} src={part.url} alt={part.filename} />;
                }

                return null;
              })}
            </div>
          </div>
        ))}
      </div>

      <form
        onSubmit={event => {
          event.preventDefault();
          if (input.trim()) {
            sendMessage({
              text: input,
              files,
            });
            setInput('');
          }
        }}
      >
        <input
          value={input}
          placeholder="Send message..."
          onChange={e => setInput(e.target.value)}
          disabled={status !== 'ready'}
        />
      </form>
    </div>
  );
}
```

## Type Inference for Tools

TypeScript에서 툴을 사용할 때, AI SDK UI는 입력/출력 타입 안전성을 보장하는 타입 추론 헬퍼를 제공해.

### InferUITool

`InferUITool` 타입 헬퍼는 단일 툴의 입력/출력 타입을 UI 메시지에서 사용하기 좋게 추론해:

```tsx
import { InferUITool } from 'ai';
import { z } from 'zod';

const weatherTool = {
  description: 'Get the current weather',
  inputSchema: z.object({
    location: z.string().describe('The city and state'),
  }),
  execute: async ({ location }) => {
    return `The weather in ${location} is sunny.`;
  },
};

// Infer the types from the tool
type WeatherUITool = InferUITool<typeof weatherTool>;
// This creates a type with:
// {
//   input: { location: string };
//   output: string;
// }
```

### InferUITools

`InferUITools` 타입 헬퍼는 `ToolSet`의 입력/출력 타입을 추론해:

```tsx
import { InferUITools, ToolSet } from 'ai';
import { z } from 'zod';

const tools = {
  weather: {
    description: 'Get the current weather',
    inputSchema: z.object({
      location: z.string().describe('The city and state'),
    }),
    execute: async ({ location }) => {
      return `The weather in ${location} is sunny.`;
    },
  },
  calculator: {
    description: 'Perform basic arithmetic',
    inputSchema: z.object({
      operation: z.enum(['add', 'subtract', 'multiply', 'divide']),
      a: z.number(),
      b: z.number(),
    }),
    execute: async ({ operation, a, b }) => {
      switch (operation) {
        case 'add':
          return a + b;
        case 'subtract':
          return a - b;
        case 'multiply':
          return a * b;
        case 'divide':
          return a / b;
      }
    },
  },
} satisfies ToolSet;

// Infer the types from the tool set
type MyUITools = InferUITools<typeof tools>;
// This creates a type with:
// {
//   weather: { input: { location: string }; output: string };
//   calculator: { input: { operation: 'add' | 'subtract' | 'multiply' | 'divide'; a: number; b: number }; output: number };
// }
```

### Using Inferred Types

추론된 타입으로 커스텀 UIMessage 타입을 만들고, 다양한 AI SDK UI 함수에 전달해 사용할 수 있어:

```tsx
import { InferUITools, UIMessage, UIDataTypes } from 'ai';

type MyUITools = InferUITools<typeof tools>;
type MyUIMessage = UIMessage<never, UIDataTypes, MyUITools>;
```

`useChat`이나 `createUIMessageStream`에 커스텀 타입을 넘겨서 사용:

```tsx
import { useChat } from '@ai-sdk/react';
import { createUIMessageStream } from 'ai';
import type { MyUIMessage } from './types';

// With useChat
const { messages } = useChat<MyUIMessage>();

// With createUIMessageStream
const stream = createUIMessageStream<MyUIMessage>(/* ... */);
```

이렇게 하면 클라이언트/서버 모두에서 툴 입출력에 대한 완전한 타입 안전성을 확보할 수 있어.
