# Error Handling and warnings

## Warnings

AI SDK는 예상대로 동작하지 않을 수 있는 상황에서 경고를 보여줘. 이 경고들은 실제 에러가 나기 전에 문제를 고치도록 도와줘.

### 경고가 표시되는 시점

브라우저 콘솔에 아래 상황에서 경고가 표시돼:

- 지원하지 않는 설정: 사용하는 AI 모델이 지원하지 않는 설정을 썼을 때
- 지원하지 않는 툴: 사용하는 AI 모델이 사용할 수 없는 툴을 썼을 때
- 기타 이슈: AI 모델이 다른 문제를 보고했을 때

### 경고 메시지 형식

모든 경고는 쉽게 찾을 수 있도록 "AI SDK Warning:"으로 시작해. 예를 들어:

```
AI SDK Warning: The "temperature" setting is not supported by this model
AI SDK Warning: The tool "calculator" is not supported by this model
```

### 경고 끄기

기본적으로 경고는 콘솔에 표시돼. 이 동작은 아래처럼 제어할 수 있어.

#### 모든 경고 끄기

전역 변수를 설정해 경고를 완전히 끌 수 있어:

```ts
globalThis.AI_SDK_LOG_WARNINGS = false;
```

#### 커스텀 경고 핸들러

경고를 직접 처리하는 함수를 제공할 수도 있어:

```ts
globalThis.AI_SDK_LOG_WARNINGS = warnings => {
  // 네 방식대로 경고 처리
  warnings.forEach(warning => {
    // 커스텀 로직
    console.log('Custom warning:', warning);
  });
};
```

<Note>
  커스텀 경고 함수는 실험적이야. 공지 없이 패치 릴리스에서 변경될 수 있어.
</Note>

## Error Handling

### Error Helper Object

각 AI SDK UI 훅은 UI에 에러를 렌더링할 수 있도록 [error](/docs/reference/ai-sdk-ui/use-chat#error) 객체를 반환해.
이 에러 객체를 사용해서 에러 메시지를 보여주거나, 제출 버튼을 비활성화하거나, 재시도 버튼을 보여줄 수 있어.

<Note>
  사용자에게는 "문제가 발생했어." 같은 일반적인 에러 메시지를 보여주는 걸 권장해. 서버 정보 노출을 피할 수 있어.
</Note>

```tsx file="app/page.tsx" highlight="7,18-25,31"
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export default function Chat() {
  const [input, setInput] = useState('');
  const { messages, sendMessage, error, regenerate } = useChat();

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    sendMessage({ text: input });
    setInput('');
  };

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.role}:{' '}
          {m.parts
            .filter(part => part.type === 'text')
            .map(part => part.text)
            .join('')}
        </div>
      ))}

      {error && (
        <>
          <div>An error occurred.</div>
          <button type="button" onClick={() => regenerate()}>
            Retry
          </button>
        </>
      )}

      <form onSubmit={handleSubmit}>
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

#### 대안: 마지막 메시지 교체

에러가 있을 때 마지막 메시지를 교체하는 커스텀 제출 핸들러를 직접 작성할 수도 있어.

```tsx file="app/page.tsx" highlight="17-23,35"
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export default function Chat() {
  const [input, setInput] = useState('');
  const { sendMessage, error, messages, setMessages } = useChat();

  function customSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();

    if (error != null) {
      setMessages(messages.slice(0, -1)); // remove last message
    }

    sendMessage({ text: input });
    setInput('');
  }

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.role}:{' '}
          {m.parts
            .filter(part => part.type === 'text')
            .map(part => part.text)
            .join('')}
        </div>
      ))}

      {error && <div>An error occurred.</div>}

      <form onSubmit={customSubmit}>
        <input value={input} onChange={e => setInput(e.target.value)} />
      </form>
    </div>
  );
}
```

### Error Handling Callback

에러는 [`useChat`](/docs/reference/ai-sdk-ui/use-chat) 또는 [`useCompletion`](/docs/reference/ai-sdk-ui/use-completion) 훅 옵션에 [`onError`](/docs/reference/ai-sdk-ui/use-chat#on-error) 콜백을 전달해서 처리할 수 있어.
콜백 함수는 에러 객체를 인자로 받아.

```tsx file="app/page.tsx" highlight="6-9"
import { useChat } from '@ai-sdk/react';

export default function Page() {
  const {
    /* ... */
  } = useChat({
    // handle error:
    onError: error => {
      console.error(error);
    },
  });
}
```

### 테스트를 위한 에러 주입

테스트 목적으로 에러를 의도적으로 만들고 싶을 수 있어.
라우트 핸들러에서 에러를 던지면 간단히 재현할 수 있어:

```ts file="app/api/chat/route.ts"
export async function POST(req: Request) {
  throw new Error('This is a test error');
}
```
