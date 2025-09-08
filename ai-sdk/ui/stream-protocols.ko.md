# Stream Protocols

AI SDK UI의 `useChat`, `useCompletion` 같은 함수는 텍스트 스트림과 데이터 스트림을 모두 지원해.
스트림 프로토콜은 HTTP 위에서 프론트엔드로 데이터가 어떻게 스트리밍되는지를 정의해.

이 페이지는 두 프로토콜과, 백엔드/프론트엔드에서 어떻게 사용하는지 설명해.

이 정보를 바탕으로 너의 사용 사례에 맞는 커스텀 백엔드/프론트엔드를 만들 수 있어. 예를 들어 Python 같은 다른 언어로 구현된 호환 API 엔드포인트를 제공할 수 있어.

예시로, 백엔드로 [FastAPI](https://github.com/vercel/ai/tree/main/examples/next-fastapi)를 쓰는 경우가 있어.

## Text Stream Protocol

텍스트 스트림은 일반 텍스트 청크로 구성되고, 프론트엔드로 스트리밍돼.
각 청크는 이어 붙여져서 최종 텍스트 응답을 만들어.

텍스트 스트림은 `useChat`, `useCompletion`, `useObject`에서 지원돼.
`useChat`이나 `useCompletion`을 사용할 때는 `streamProtocol` 옵션을 `text`로 설정해 텍스트 스트리밍을 켜야 해.

백엔드에서는 `streamText`로 텍스트 스트림을 생성할 수 있어.
결과 객체에서 `toTextStreamResponse()`를 호출하면 스트리밍 HTTP 응답이 반환돼.

<Note>
  텍스트 스트림은 기본 텍스트 데이터만 지원해. 툴 콜 같은 다른 타입의 데이터를 스트리밍하려면 데이터 스트림을 사용해.
</Note>

### Text Stream Example

다음은 텍스트 스트림 프로토콜을 사용하는 Next.js 예시야:

```tsx filename='app/page.tsx'
'use client';

import { useChat } from '@ai-sdk/react';
import { TextStreamChatTransport } from 'ai';
import { useState } from 'react';

export default function Chat() {
  const [input, setInput] = useState('');
  const { messages, sendMessage } = useChat({
    transport: new TextStreamChatTransport({ api: '/api/chat' }),
  });

  return (
    <div className="flex flex-col w-full max-w-md py-24 mx-auto stretch">
      {messages.map(message => (
        <div key={message.id} className="whitespace-pre-wrap">
          {message.role === 'user' ? 'User: ' : 'AI: '}
          {message.parts.map((part, i) => {
            switch (part.type) {
              case 'text':
                return <div key={`${message.id}-${i}`}>{part.text}</div>;
            }
          })}
        </div>
      ))}

      <form
        onSubmit={e => {
          e.preventDefault();
          sendMessage({ text: input });
          setInput('');
        }}
      >
        <input
          className="fixed dark:bg-zinc-900 bottom-0 w-full max-w-md p-2 mb-8 border border-zinc-300 dark:border-zinc-800 rounded shadow-xl"
          value={input}
          placeholder="Say something..."
          onChange={e => setInput(e.currentTarget.value)}
        />
      </form>
    </div>
  );
}
```

```ts filename='app/api/chat/route.ts'
import { streamText, UIMessage, convertToModelMessages } from 'ai';
import { openai } from '@ai-sdk/openai';

// Allow streaming responses up to 30 seconds
export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages: convertToModelMessages(messages),
  });

  return result.toTextStreamResponse();
}
```

## Data Stream Protocol

데이터 스트림은 프론트엔드로 정보를 보내기 위해 AI SDK가 제공하는 특별한 프로토콜을 따르며,
표준화, ping 기반 keep-alive, 재연결, 더 나은 캐시 처리를 위해 Server-Sent Events(SSE) 형식을 사용해.

<Note>
  커스텀 백엔드에서 데이터 스트림을 제공할 때는
  `x-vercel-ai-ui-message-stream` 헤더를 `v1`로 설정해야 해.
</Note>

현재 지원되는 스트림 파트는 다음과 같아:

### Message Start Part

메타데이터와 함께 새 메시지의 시작을 나타내.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"start","messageId":"..."}

```

### Text Parts

텍스트 콘텐츠는 각 텍스트 블록에 대해 고유 ID와 함께 start/delta/end 패턴으로 스트리밍돼.

#### Text Start Part

텍스트 블록의 시작을 나타내.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"text-start","id":"msg_68679a454370819ca74c8eb3d04379630dd1afb72306ca5d"}

```

#### Text Delta Part

텍스트 블록의 증분 텍스트 내용을 담아.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"text-delta","id":"msg_68679a454370819ca74c8eb3d04379630dd1afb72306ca5d","delta":"Hello"}

```

#### Text End Part

텍스트 블록의 완료를 나타내.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"text-end","id":"msg_68679a454370819ca74c8eb3d04379630dd1afb72306ca5d"}

```

### Reasoning Parts

리저닝 콘텐츠는 각 리저닝 블록에 대해 start/delta/end 패턴으로 스트리밍돼.

#### Reasoning Start Part

리저닝 블록의 시작을 나타내.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"reasoning-start","id":"reasoning_123"}

```

#### Reasoning Delta Part

리저닝 블록의 증분 내용을 담아.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"reasoning-delta","id":"reasoning_123","delta":"This is some reasoning"}

```

#### Reasoning End Part

리저닝 블록의 완료를 나타내.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"reasoning-end","id":"reasoning_123"}

```

### Source Parts

외부 콘텐츠 출처에 대한 레퍼런스를 제공해.

#### Source URL Part

외부 URL 레퍼런스.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"source-url","sourceId":"https://example.com","url":"https://example.com"}

```

#### Source Document Part

문서/파일에 대한 레퍼런스.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"source-document","sourceId":"https://example.com","mediaType":"file","title":"Title"}

```

### File Part

파일 파트는 파일 레퍼런스와 그 미디어 타입을 담아.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"file","url":"https://example.com/file.png","mediaType":"image/png"}

```

### Data Parts

커스텀 데이터 파트를 사용하면 타입별 처리와 함께 임의의 구조화 데이터를 스트리밍할 수 있어.

형식: 타입에 커스텀 접미사가 포함된 JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"data-weather","data":{"location":"SF","temperature":100}}

```

`data-*` 타입 패턴을 사용하면 프론트엔드가 특정하게 처리할 수 있는 커스텀 데이터 타입을 정의할 수 있어.

### Error Part

에러 파트는 수신되는 대로 메시지에 추가돼.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"error","errorText":"error message"}

```

### Tool Input Start Part

툴 입력 스트리밍의 시작을 나타내.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"tool-input-start","toolCallId":"call_fJdQDqnXeGxTmr4E3YPSR7Ar","toolName":"getWeatherInformation"}

```

### Tool Input Delta Part

생성 중인 툴 입력의 증분 청크를 담아.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"tool-input-delta","toolCallId":"call_fJdQDqnXeGxTmr4E3YPSR7Ar","inputTextDelta":"San Francisco"}

```

### Tool Input Available Part

툴 입력이 완료되어 실행 준비가 되었음을 나타내.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"tool-input-available","toolCallId":"call_fJdQDqnXeGxTmr4E3YPSR7Ar","toolName":"getWeatherInformation","input":{"city":"San Francisco"}}

```

### Tool Output Available Part

툴 실행 결과를 담아.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"tool-output-available","toolCallId":"call_fJdQDqnXeGxTmr4E3YPSR7Ar","output":{"city":"San Francisco","weather":"sunny"}}

```

### Start Step Part

스텝의 시작을 나타내는 파트야.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"start-step"}

```

### Finish Step Part

스텝(즉, 백엔드에서의 한 번의 LLM API 호출)이 완료되었음을 나타내는 파트야.

이 파트는 백엔드에서 툴을 호출하면서 동시에 `useChat`에서 스텝을 사용할 때처럼,
여러 어시스턴트 호출을 이어 붙여 정확히 처리하기 위해 필요해.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"finish-step"}

```

### Finish Message Part

메시지의 완료를 나타내는 파트야.

형식: JSON 객체를 담은 Server-Sent Event

예시:

```
data: {"type":"finish"}

```

### Stream Termination

스트림은 특별한 `[DONE]` 마커로 끝나.

형식: 리터럴 `[DONE]`을 담은 Server-Sent Event

예시:

```
data: [DONE]

```

데이터 스트림 프로토콜은 프론트엔드의 `useChat`, `useCompletion`에서 지원되며 기본값이야.
`useCompletion`은 `text`, `data` 스트림 파트만 지원해.

백엔드에선 `streamText` 결과 객체의 `toUIMessageStreamResponse()`를 사용해 스트리밍 HTTP 응답을 반환할 수 있어.

### UI Message Stream Example

다음은 UI 메시지 스트림 프로토콜을 사용하는 Next.js 예시야:

```tsx filename='app/page.tsx'
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export default function Chat() {
  const [input, setInput] = useState('');
  const { messages, sendMessage } = useChat();

  return (
    <div className="flex flex-col w-full max-w-md py-24 mx-auto stretch">
      {messages.map(message => (
        <div key={message.id} className="whitespace-pre-wrap">
          {message.role === 'user' ? 'User: ' : 'AI: '}
          {message.parts.map((part, i) => {
            switch (part.type) {
              case 'text':
                return <div key={`${message.id}-${i}`}>{part.text}</div>;
            }
          })}
        </div>
      ))}

      <form
        onSubmit={e => {
          e.preventDefault();
          sendMessage({ text: input });
          setInput('');
        }}
      >
        <input
          className="fixed dark:bg-zinc-900 bottom-0 w-full max-w-md p-2 mb-8 border border-zinc-300 dark:border-zinc-800 rounded shadow-xl"
          value={input}
          placeholder="Say something..."
          onChange={e => setInput(e.currentTarget.value)}
        />
      </form>
    </div>
  );
}
```

```ts filename='app/api/chat/route.ts'
import { openai } from '@ai-sdk/openai';
import { streamText, UIMessage, convertToModelMessages } from 'ai';

// Allow streaming responses up to 30 seconds
export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse();
}
```

---
