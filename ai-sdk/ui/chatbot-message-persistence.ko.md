# 챗봇 메시지 영속성(Chatbot Message Persistence)

대부분의 AI 챗봇에선 대화를 저장하고 다시 불러올 수 있어야 합니다.
이 가이드에서는 `useChat`과 `streamText`를 사용해 메시지 영속성을 구현하는 방법을 설명합니다.

<Note>
  이 가이드는 인증, 에러 처리 같은 실무 고려사항을 다루지 않습니다.
  메시지 영속성을 구현하는 간단한 예제로 의도되었습니다.
</Note>

## 새 채팅 시작하기

사용자가 채팅 ID 없이 채팅 페이지로 이동하면,
새 채팅을 생성한 뒤 해당 채팅 ID로 리디렉션해야 합니다.

```tsx filename="app/chat/page.tsx"
import { redirect } from 'next/navigation';
import { createChat } from '@util/chat-store';

export default async function Page() {
  const id = await createChat(); // create a new chat
  redirect(`/chat/${id}`); // redirect to chat page, see below
}
```

예시 채팅 저장소 구현은 파일을 이용해 채팅 메시지를 저장합니다.
실제 애플리케이션에서는 데이터베이스나 클라우드 스토리지를 사용해
채팅 ID를 발급받는 것이 일반적입니다.
함수 인터페이스는 다른 구현으로 쉽게 교체할 수 있도록 설계되어 있습니다.

```tsx filename="util/chat-store.ts"
import { generateId } from 'ai';
import { existsSync, mkdirSync } from 'fs';
import { writeFile } from 'fs/promises';
import path from 'path';

export async function createChat(): Promise<string> {
  const id = generateId(); // generate a unique chat ID
  await writeFile(getChatFile(id), '[]'); // create an empty chat file
  return id;
}

function getChatFile(id: string): string {
  const chatDir = path.join(process.cwd(), '.chats');
  if (!existsSync(chatDir)) mkdirSync(chatDir, { recursive: true });
  return path.join(chatDir, `${id}.json`);
}
```

## 기존 채팅 불러오기

사용자가 채팅 ID와 함께 채팅 페이지로 이동하면 저장소에서 채팅 메시지를 불러와야 합니다.

파일 기반 채팅 저장소의 `loadChat` 함수는 다음과 같습니다:

```tsx filename="util/chat-store.ts"
import { UIMessage } from 'ai';
import { readFile } from 'fs/promises';

export async function loadChat(id: string): Promise<UIMessage[]> {
  return JSON.parse(await readFile(getChatFile(id), 'utf8'));
}

// ... rest of the file
```

## 서버에서 메시지 검증하기

툴 호출, 커스텀 메타데이터, 데이터 파트가 포함된 서버 측 메시지를 처리할 때는
모델에 전달하기 전에 `validateUIMessages`로 검증해야 합니다.

### 툴과 함께 검증하기

메시지에 툴 호출이 포함되어 있다면, 정의한 툴 스키마에 대해 검증하세요:

```tsx filename="app/api/chat/route.ts" highlight="7-25,32-37"
import { convertToModelMessages, streamText, UIMessage, validateUIMessages, tool } from 'ai';
import { z } from 'zod';
import { loadChat, saveChat } from '@util/chat-store';
import { openai } from '@ai-sdk/openai';
import { dataPartsSchema, metadataSchema } from '@util/schemas';

// Define your tools
const tools = {
  weather: tool({
    description: 'Get weather information',
    parameters: z.object({
      location: z.string(),
      units: z.enum(['celsius', 'fahrenheit']),
    }),
    execute: async ({ location, units }) => {
      /* tool implementation */
    },
  }),
  // other tools
};

export async function POST(req: Request) {
  const { message, id } = await req.json();

  // Load previous messages from database
  const previousMessages = await loadChat(id);

  // Append new message to previousMessages messages
  const messages = [...previousMessages, message];

  // Validate loaded messages against
  // tools, data parts schema, and metadata schema
  const validatedMessages = await validateUIMessages({
    messages,
    tools, // Ensures tool calls in messages match current schemas
    dataPartsSchema,
    metadataSchema,
  });

  const result = streamText({
    model: openai('gpt-4o-mini'),
    messages: convertToModelMessages(validatedMessages),
    tools,
  });

  return result.toUIMessageStreamResponse({
    originalMessages: messages,
    onFinish: ({ messages }) => {
      saveChat({ chatId: id, messages });
    },
  });
}
```

### 검증 에러 처리

데이터베이스의 메시지가 현재 스키마와 맞지 않을 때는 에러를 무해하게 처리하세요:

```tsx filename="app/api/chat/route.ts" highlight="3,10-24"
import { convertToModelMessages, streamText, validateUIMessages, TypeValidationError } from 'ai';
import { type MyUIMessage } from '@/types';

export async function POST(req: Request) {
  const { message, id } = await req.json();

  // Load and validate messages from database
  let validatedMessages: MyUIMessage[];

  try {
    const previousMessages = await loadMessagesFromDB(id);
    validatedMessages = await validateUIMessages({
      // append the new message to the previous messages:
      messages: [...previousMessages, message],
      tools,
      metadataSchema,
    });
  } catch (error) {
    if (error instanceof TypeValidationError) {
      // Log validation error for monitoring
      console.error('Database messages validation failed:', error);
      // Could implement message migration or filtering here
      // For now, start with empty history
      validatedMessages = [];
    } else {
      throw error;
    }
  }

  // Continue with validated messages...
}
```

## 채팅 표시하기

저장소에서 메시지를 불러왔으면, 채팅 UI에 표시할 수 있습니다. 페이지 컴포넌트와 표시 예시는 다음과 같습니다:

```tsx filename="app/chat/[id]/page.tsx"
import { loadChat } from '@util/chat-store';
import Chat from '@ui/chat';

export default async function Page(props: { params: Promise<{ id: string }> }) {
  const { id } = await props.params;
  const messages = await loadChat(id);
  return (
    <Chat
      id={id}
      initialMessages={messages}
    />
  );
}
```

채팅 컴포넌트는 `useChat` 훅을 사용해 대화를 관리합니다:

```tsx filename="ui/chat.tsx" highlight="10-16"
'use client';

import { UIMessage, useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';
import { useState } from 'react';

export default function Chat({
  id,
  initialMessages,
}: { id?: string | undefined; initialMessages?: UIMessage[] } = {}) {
  const [input, setInput] = useState('');
  const { sendMessage, messages } = useChat({
    id, // use the provided chat ID
    messages: initialMessages, // load initial messages
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (input.trim()) {
      sendMessage({ text: input });
      setInput('');
    }
  };

  // simplified rendering code, extend as needed:
  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>
          {m.role === 'user' ? 'User: ' : 'AI: '}
          {m.parts.map((part) => (part.type === 'text' ? part.text : '')).join('')}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Type a message..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

## 메시지 저장하기

`useChat`은 채팅 ID와 메시지를 백엔드로 전송합니다.

<Note>
  `useChat` 메시지 포맷은 `ModelMessage` 포맷과 다릅니다.
  `useChat` 메시지 포맷은 프런트엔드 표시를 위해 설계되었고,
  `id`, `createdAt` 같은 필드를 추가로 포함합니다. 저장 포맷으로 `useChat` 메시지 포맷을 권장합니다.

툴, 메타데이터, 커스텀 데이터 파트가 포함된 메시지를 저장소에서 불러올 때는
처리 전에 `validateUIMessages`로 검증하세요(위의 [검증 섹션](#validating-messages-from-database) 참고).
</Note>

메시지 저장은 `toUIMessageStreamResponse`의 `onFinish` 콜백에서 수행합니다.
`onFinish`는 새로운 AI 응답을 포함한 완전한 `UIMessage[]`를 전달합니다.

```tsx filename="app/api/chat/route.ts" highlight="6,11-17"
import { openai } from '@ai-sdk/openai';
import { saveChat } from '@util/chat-store';
import { convertToModelMessages, streamText, UIMessage } from 'ai';

export async function POST(req: Request) {
  const { messages, chatId }: { messages: UIMessage[]; chatId: string } = await req.json();

  const result = streamText({
    model: openai('gpt-4o-mini'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse({
    originalMessages: messages,
    onFinish: ({ messages }) => {
      saveChat({ chatId, messages });
    },
  });
}
```

실제 저장 동작은 파일 기반 저장소의 `saveChat` 함수가 담당합니다:

```tsx filename="util/chat-store.ts"
import { UIMessage } from 'ai';
import { writeFile } from 'fs/promises';

export async function saveChat({
  chatId,
  messages,
}: {
  chatId: string;
  messages: UIMessage[];
}): Promise<void> {
  const content = JSON.stringify(messages, null, 2);
  await writeFile(getChatFile(chatId), content);
}

// ... rest of the file
```

## 메시지 ID

채팅 ID 외에도 각 메시지는 고유한 ID를 가집니다.
이 메시지 ID를 이용해 개별 메시지를 조작할 수 있습니다.

### 클라이언트 vs 서버 측 ID 생성

기본적으로 메시지 ID는 클라이언트에서 생성됩니다:

- 사용자 메시지 ID는 클라이언트의 `useChat` 훅이 생성합니다
- AI 응답 메시지 ID는 서버의 `streamText`가 생성합니다

영속성이 필요 없는 애플리케이션에서는 클라이언트 측 ID 생성만으로 충분합니다.
하지만 영속성이 필요하다면, 세션 간 일관성을 보장하고 저장/복원 시 ID 충돌을 방지하기 위해
**서버 측에서 생성된 ID**가 필요합니다.

### 서버 측 ID 생성 설정

영속성을 구현할 때 서버 측 ID를 생성하는 방법은 두 가지가 있습니다:

1. `toUIMessageStreamResponse`의 `generateMessageId` 사용
2. `createUIMessageStream`으로 시작 메시지 파트에 ID 설정

#### 옵션 1: `toUIMessageStreamResponse`의 `generateMessageId` 사용

[`createIdGenerator()`](/docs/reference/ai-sdk-core/create-id-generator)를 이용해 ID 생성기를 제공하여 ID 형식을 제어할 수 있습니다:

```tsx filename="app/api/chat/route.ts" highlight="7-11"
import { createIdGenerator, streamText } from 'ai';

export async function POST(req: Request) {
  // ...
  const result = streamText({
    // ...
  });

  return result.toUIMessageStreamResponse({
    originalMessages: messages,
    // Generate consistent server-side IDs for persistence:
    generateMessageId: createIdGenerator({
      prefix: 'msg',
      size: 16,
    }),
    onFinish: ({ messages }) => {
      saveChat({ chatId, messages });
    },
  });
}
```

#### 옵션 2: `createUIMessageStream`으로 ID 설정

`createUIMessageStream`을 사용해 시작 메시지 파트를 작성하면서 메시지 ID를 제어할 수 있습니다:

```tsx filename="app/api/chat/route.ts" highlight="8-18"
import { generateId, streamText, createUIMessageStream, createUIMessageStreamResponse } from 'ai';

export async function POST(req: Request) {
  const { messages, chatId } = await req.json();

  const stream = createUIMessageStream({
    execute: ({ writer }) => {
      // Write start message part with custom ID
      writer.write({
        type: 'start',
        messageId: generateId(), // Generate server-side ID for persistence
      });

      const result = streamText({
        model: openai('gpt-4o-mini'),
        messages: convertToModelMessages(messages),
      });

      writer.merge(result.toUIMessageStream({ sendStart: false })); // omit start message part
    },
    originalMessages: messages,
    onFinish: ({ responseMessage }) => {
      // save your chat here
    },
  });

  return createUIMessageStreamResponse({ stream });
}
```

<Note>
  영속성이 필요 없는 클라이언트 측 애플리케이션에서도 클라이언트 측 ID 생성을 커스터마이즈할 수 있습니다:

```tsx filename="ui/chat.tsx"
import { createIdGenerator } from 'ai';
import { useChat } from '@ai-sdk/react';

const { ... } = useChat({
  generateId: createIdGenerator({
    prefix: 'msgc',
    size: 16,
  }),
  // ...
});
```

</Note>

## 마지막 메시지만 보내기

메시지 영속성을 구현했다면, 매 요청마다 마지막 메시지만 서버로 보내고 싶을 수 있습니다.
이는 서버로 전송되는 데이터 양을 줄여 성능을 개선할 수 있습니다.

이를 위해 transport에 `prepareSendMessagesRequest` 함수를 제공하면 됩니다.
이 함수는 메시지와 채팅 ID를 입력받아 서버로 보낼 요청 본문을 반환합니다.

```tsx filename="ui/chat.tsx" highlight="7-12"
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';

const {
  // ...
} = useChat({
  // ...
  transport: new DefaultChatTransport({
    api: '/api/chat',
    // only send the last message to the server:
    prepareSendMessagesRequest({ messages, id }) {
      return { body: { message: messages[messages.length - 1], id } };
    },
  }),
});
```

서버에서는 이전 메시지를 로드해 새 메시지를 뒤에 붙이면 됩니다. 메시지에 툴, 메타데이터, 커스텀 데이터 파트가 있다면 검증하세요:

```tsx filename="app/api/chat/route.ts" highlight="2-11,14-18"
import { convertToModelMessages, UIMessage, validateUIMessages } from 'ai';
// import your tools and schemas

export async function POST(req: Request) {
  // get the last message from the client:
  const { message, id } = await req.json();

  // load the previous messages from the server:
  const previousMessages = await loadChat(id);

  // validate messages if they contain tools, metadata, or data parts:
  const validatedMessages = await validateUIMessages({
    // append the new message to the previous messages:
    messages: [...previousMessages, message],
    tools, // if using tools
    metadataSchema, // if using custom metadata
    dataSchemas, // if using custom data parts
  });

  const result = streamText({
    // ...
    messages: convertToModelMessages(validatedMessages),
  });

  return result.toUIMessageStreamResponse({
    originalMessages: validatedMessages,
    onFinish: ({ messages }) => {
      saveChat({ chatId: id, messages });
    },
  });
}
```

## 클라이언트 연결 해제 처리

기본적으로 AI SDK의 `streamText`는 백프레셔를 사용해
아직 요청되지 않은 토큰 소비를 방지합니다.

하지만 이는 사용자가 브라우저 탭을 닫거나 네트워크 이슈로
클라이언트가 연결 해제되면 LLM의 스트림이 중단되고
대화가 깨진 상태로 남을 수 있음을 의미합니다.

[저장소 솔루션](#storing-messages)을 사용 중이라고 가정하면,
백엔드에서 `consumeStream`을 호출해 스트림을 끝까지 소비한 뒤
결과를 평소처럼 저장할 수 있습니다.
`consumeStream`은 사실상 백프레셔를 제거하여,
클라이언트 응답이 이미 중단되었더라도 결과가 저장되게 합니다.

```tsx filename="app/api/chat/route.ts" highlight="19-21"
import { convertToModelMessages, streamText, UIMessage } from 'ai';
import { saveChat } from '@util/chat-store';

export async function POST(req: Request) {
  const { messages, chatId }: { messages: UIMessage[]; chatId: string } = await req.json();

  const result = streamText({
    model,
    messages: convertToModelMessages(messages),
  });

  // consume the stream to ensure it runs to completion & triggers onFinish
  // even when the client response is aborted:
  result.consumeStream(); // no await

  return result.toUIMessageStreamResponse({
    originalMessages: messages,
    onFinish: ({ messages }) => {
      saveChat({ chatId, messages });
    },
  });
}
```

클라이언트가 연결 해제 후 페이지를 새로고침하면, 저장소 솔루션에서 채팅이 복원됩니다.

<Note>
  프로덕션 애플리케이션에서는 요청 상태(진행 중, 완료)를 저장된 메시지에 함께 추적해
  클라이언트가 연결 해제 후 페이지를 새로고침했는데도 스트리밍이 아직 끝나지 않은
  경우를 커버해야 합니다.
</Note>

연결 해제를 보다 견고하게 처리하려면 재개 기능(resumability)을 추가할 수 있습니다. 자세한 내용은 [Chatbot Resume Streams](/docs/ai-sdk-ui/chatbot-resume-streams) 문서를 참고하세요.
