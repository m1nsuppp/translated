# Error Handling (에러 처리)

## 일반 에러 처리

일반 에러는 throw 되고, `try/catch` 블록으로 처리하면 돼.

```ts highlight="3,8-10"
import { generateText } from 'ai';

try {
  const { text } = await generateText({
    model: 'openai/gpt-4.1',
    prompt: 'Write a vegetarian lasagna recipe for 4 people.',
  });
} catch (error) {
  // handle error
}
```

던져질 수 있는 다양한 에러 타입은 [Error Types](/docs/reference/ai-sdk-errors) 문서를 참고해.

## 스트리밍 에러 처리(단순 스트림)

에러 청크를 지원하지 않는 스트림 중에 에러가 발생하면, 일반 에러처럼 throw 돼.
`try/catch` 블록으로 처리하면 돼.

```ts highlight="3,12-14"
import { generateText } from 'ai';

try {
  const { textStream } = streamText({
    model: 'openai/gpt-4.1',
    prompt: 'Write a vegetarian lasagna recipe for 4 people.',
  });

  for await (const textPart of textStream) {
    process.stdout.write(textPart);
  }
} catch (error) {
  // handle error
}
```

## 스트리밍 에러 처리(`error` 지원 스트림)

풀 스트림은 에러 파트를 지원해. 다른 파트들처럼 처리하면 돼.
스트리밍 외부에서 발생하는 에러를 위해 `try/catch`도 같이 쓰는 걸 추천해.

```ts highlight="13-21"
import { generateText } from 'ai';

try {
  const { fullStream } = streamText({
    model: 'openai/gpt-4.1',
    prompt: 'Write a vegetarian lasagna recipe for 4 people.',
  });

  for await (const part of fullStream) {
    switch (part.type) {
      // ... handle other part types

      case 'error': {
        const error = part.error;
        // handle error
        break;
      }

      case 'abort': {
        // handle stream abort
        break;
      }

      case 'tool-error': {
        const error = part.error;
        // handle error
        break;
      }
    }
  }
} catch (error) {
  // handle error
}
```

## 스트림 중단(abort) 처리

스트림이 중단될 때(예: 채팅 중지 버튼), UI에 저장된 메시지 업데이트 같은 정리 작업이 필요할 수 있어. 이런 경우 `onAbort` 콜백을 사용해.

`onAbort`는 `AbortSignal`로 스트림이 중단되면 호출되고, `onFinish`는 호출되지 않아. 덕분에 UI 상태를 적절히 갱신할 수 있어.

```ts highlight="5-9"
import { streamText } from 'ai';

const { textStream } = streamText({
  model: 'openai/gpt-4.1',
  prompt: 'Write a vegetarian lasagna recipe for 4 people.',
  onAbort: ({ steps }) => {
    // 저장된 메시지 업데이트 등 정리 작업
    console.log('Stream aborted after', steps.length, 'steps');
  },
  onFinish: ({ steps, totalUsage }) => {
    // 정상 완료 시 호출
    console.log('Stream completed normally');
  },
});

for await (const textPart of textStream) {
  process.stdout.write(textPart);
}
```

`onAbort` 콜백이 받는 값:

- `steps`: 중단 이전까지 완료된 모든 스텝 배열

스트림 안에서 직접 abort 이벤트를 처리할 수도 있어:

```ts highlight="10-13"
import { streamText } from 'ai';

const { fullStream } = streamText({
  model: 'openai/gpt-4.1',
  prompt: 'Write a vegetarian lasagna recipe for 4 people.',
});

for await (const chunk of fullStream) {
  switch (chunk.type) {
    case 'abort': {
      // 스트림 중단 직접 처리
      console.log('Stream was aborted');
      break;
    }
    // ... handle other part types
  }
}
```
