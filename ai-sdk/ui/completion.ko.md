# Completion

`useCompletion` 훅은 앱에서 텍스트 컴플리션을 처리하는 UI를 만들 수 있게 해줘. AI 제공자로부터 컴플리션을 스트리밍으로 받아오고, 채팅 입력 상태를 관리하며, 새 메시지가 도착할 때 UI를 자동으로 업데이트해.

<Note>
  `useCompletion` 훅은 이제 `@ai-sdk/react` 패키지에 포함돼.
</Note>

이 가이드에서는 앱에서 `useCompletion` 훅을 사용해 텍스트 컴플리션을 생성하고, 이를 실시간으로 사용자에게 스트리밍하는 방법을 배울 수 있어.

## 예시

```tsx filename='app/page.tsx'
'use client';

import { useCompletion } from '@ai-sdk/react';

export default function Page() {
  const { completion, input, handleInputChange, handleSubmit } = useCompletion({
    api: '/api/completion',
  });

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="prompt"
        value={input}
        onChange={handleInputChange}
        id="input"
      />
      <button type="submit">Submit</button>
      <div>{completion}</div>
    </form>
  );
}
```

```ts filename='app/api/completion/route.ts'
import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

// Allow streaming responses up to 30 seconds
export const maxDuration = 30;

export async function POST(req: Request) {
  const { prompt }: { prompt: string } = await req.json();

  const result = streamText({
    model: openai('gpt-3.5-turbo'),
    prompt,
  });

  return result.toUIMessageStreamResponse();
}
```

`Page` 컴포넌트에서, 사용자가 메시지를 제출할 때마다 `useCompletion` 훅이 AI 제공자 엔드포인트로 요청을 보낼 거야. 컴플리션은 실시간으로 스트리밍되어 UI에 표시돼.

이렇게 하면 전체 응답을 다 기다리지 않고도, 사용자가 곧바로 AI의 응답을 보기 시작할 수 있어서 매끄러운 컴플리션 경험을 제공할 수 있어.

## 커스텀 UI

`useCompletion`은 코드로 프롬프트를 관리하고, 로딩/에러 상태를 표시하며, 사용자 상호작용 없이도 메시지를 업데이트하는 방법을 제공해.

### 로딩/에러 상태

사용자 메시지를 처리하는 동안 로딩 스피너를 보여주려면, `useCompletion` 훅이 반환하는 `isLoading` 상태를 쓰면 돼:

```tsx
const { isLoading, ... } = useCompletion()

return(
  <>
    {isLoading ? <Spinner /> : null}
  </>
)
```

비슷하게, `error` 상태는 fetch 요청 중 던져진 에러 객체를 반영해. 에러 메시지를 표시하거나 토스트 알림을 보여줄 때 사용할 수 있어:

```tsx
const { error, ... } = useCompletion()

useEffect(() => {
  if (error) {
    toast.error(error.message)
  }
}, [error])

// Or display the error message in the UI:
return (
  <>
    {error ? <div>{error.message}</div> : null}
  </>
)
```

### 제어된 입력(Controlled input)

처음 예시에서는 입력 변경과 폼 제출을 관리하는 `handleSubmit`과 `handleInputChange` 콜백을 사용했어. 이런 기본 API는 일반적인 케이스에 편리하지만, 폼 검증이나 커스텀 컴포넌트 같은 고급 시나리오에선 비제어(혹은 더 세밀한) API를 쓸 수도 있어.

아래 예시는 커스텀 입력/버튼 컴포넌트와 함께 `setInput` 같은 더 세밀한 API를 사용하는 방법을 보여줘:

```tsx
const { input, setInput } = useCompletion();

return (
  <>
    <MyCustomInput value={input} onChange={value => setInput(value)} />
  </>
);
```

### 취소(Cancelation)

응답이 스트리밍되는 도중에 중단하고 싶은 경우가 흔해. `useCompletion` 훅이 반환하는 `stop` 함수를 호출하면 돼.

```tsx
const { stop, isLoading, ... } = useCompletion()

return (
  <>
    <button onClick={stop} disabled={!isLoading}>Stop</button>
  </>
)
```

사용자가 "Stop" 버튼을 클릭하면 fetch 요청이 중단돼. 불필요한 리소스 소비를 줄이고 UX를 개선할 수 있어.

### UI 업데이트 스로틀링

<Note>이 기능은 현재 React에서만 사용할 수 있어.</Note>

기본적으로 `useCompletion` 훅은 새 청크를 받을 때마다 렌더를 트리거해.
`experimental_throttle` 옵션으로 UI 업데이트를 스로틀링할 수 있어.

```tsx filename="page.tsx" highlight="2-3"
const { completion, ... } = useCompletion({
  // Throttle the completion and data updates to 50ms:
  experimental_throttle: 50
})
```

## 이벤트 콜백

`useCompletion`은 챗봇 라이프사이클의 여러 단계에서 후킹할 수 있는 선택적 이벤트 콜백도 제공해. 로깅, 분석, 커스텀 UI 업데이트 등을 트리거하는 데 사용할 수 있어.

```tsx
const { ... } = useCompletion({
  onResponse: (response: Response) => {
    console.log('Received response from server:', response)
  },
  onFinish: (prompt: string, completion: string) => {
    console.log('Finished streaming completion:', completion)
  },
  onError: (error: Error) => {
    console.error('An error occurred:', error)
  },
})
```

참고로, `onResponse` 콜백에서 에러를 던지면 처리가 중단돼. 그러면 `onError` 콜백이 트리거되고 메시지가 채팅 UI에 추가되지 않아. AI 제공자로부터의 예상치 못한 응답을 처리할 때 유용해.

## 요청 옵션 구성

기본적으로 `useCompletion` 훅은 프롬프트를 요청 본문에 담아 `/api/completion` 엔드포인트로 HTTP POST 요청을 보낼 거야. `useCompletion` 훅에 추가 옵션을 전달해서 요청을 커스터마이즈할 수 있어:

```tsx
const { messages, input, handleInputChange, handleSubmit } = useCompletion({
  api: '/api/custom-completion',
  headers: {
    Authorization: 'your_token',
  },
  body: {
    user_id: '123',
  },
  credentials: 'same-origin',
});
```

이 예시에서 `useCompletion` 훅은 지정된 헤더, 추가 본문 필드, 그리고 fetch 요청에 대한 자격 증명과 함께 `/api/completion` 엔드포인트로 POST 요청을 보낼 거야. 서버에서는 이 추가 정보를 활용해 요청을 처리할 수 있어.
