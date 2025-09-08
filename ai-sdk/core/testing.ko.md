# Testing

언어 모델 테스트는 비결정적이고 호출이 느리고 비싸서 쉽지 않아.

AI SDK를 사용하는 코드를 단위 테스트할 수 있도록, AI SDK Core는 목(mock) 제공자와 테스트 헬퍼를 포함하고 있어. `ai/test`에서 다음 헬퍼들을 가져와서 쓸 수 있어:

- `MockEmbeddingModelV2`: [embedding model v2 사양](https://github.com/vercel/ai/blob/v5/packages/provider/src/embedding-model/v2/embedding-model-v2.ts)을 사용하는 임베딩 모델 목
- `MockLanguageModelV2`: [language model v2 사양](https://github.com/vercel/ai/blob/v5/packages/provider/src/language-model/v2/language-model-v2.ts)을 사용하는 언어 모델 목
- `mockId`: 증가하는 정수 ID 제공
- `mockValues`: 호출할 때마다 배열의 값을 순회해 반환. 배열이 끝나면 마지막 값을 계속 반환
- [`simulateReadableStream`](/docs/reference/ai-sdk-core/simulate-readable-stream): 지연을 포함해 읽기 가능한 스트림을 시뮬레이션

목 제공자와 테스트 헬퍼를 사용하면 실제 언어 모델 제공자를 호출하지 않고도, AI SDK의 출력을 제어해서 반복 가능하고 결정적인 방식으로 코드를 테스트할 수 있어.

## 예시

단위 테스트에서 AI Core 함수들과 함께 테스트 헬퍼를 사용할 수 있어:

### generateText

```ts
import { generateText } from 'ai';
import { MockLanguageModelV2 } from 'ai/test';

const result = await generateText({
  model: new MockLanguageModelV2({
    doGenerate: async () => ({
      finishReason: 'stop',
      usage: { inputTokens: 10, outputTokens: 20, totalTokens: 30 },
      content: [{ type: 'text', text: `Hello, world!` }],
      warnings: [],
    }),
  }),
  prompt: 'Hello, test!',
});
```

### streamText

```ts
import { streamText, simulateReadableStream } from 'ai';
import { MockLanguageModelV2 } from 'ai/test';

const result = streamText({
  model: new MockLanguageModelV2({
    doStream: async () => ({
      stream: simulateReadableStream({
        chunks: [
          { type: 'text-start', id: 'text-1' },
          { type: 'text-delta', id: 'text-1', delta: 'Hello' },
          { type: 'text-delta', id: 'text-1', delta: ', ' },
          { type: 'text-delta', id: 'text-1', delta: 'world!' },
          { type: 'text-end', id: 'text-1' },
          {
            type: 'finish',
            finishReason: 'stop',
            logprobs: undefined,
            usage: { inputTokens: 3, outputTokens: 10, totalTokens: 13 },
          },
        ],
      }),
    }),
  }),
  prompt: 'Hello, test!',
});
```

### generateObject

```ts
import { generateObject } from 'ai';
import { MockLanguageModelV2 } from 'ai/test';
import { z } from 'zod';

const result = await generateObject({
  model: new MockLanguageModelV2({
    doGenerate: async () => ({
      finishReason: 'stop',
      usage: { inputTokens: 10, outputTokens: 20, totalTokens: 30 },
      content: [{ type: 'text', text: `{"content":"Hello, world!"}` }],
      warnings: [],
    }),
  }),
  schema: z.object({ content: z.string() }),
  prompt: 'Hello, test!',
});
```

### streamObject

```ts
import { streamObject, simulateReadableStream } from 'ai';
import { MockLanguageModelV2 } from 'ai/test';
import { z } from 'zod';

const result = streamObject({
  model: new MockLanguageModelV2({
    doStream: async () => ({
      stream: simulateReadableStream({
        chunks: [
          { type: 'text-start', id: 'text-1' },
          { type: 'text-delta', id: 'text-1', delta: '{ ' },
          { type: 'text-delta', id: 'text-1', delta: '"content": ' },
          { type: 'text-delta', id: 'text-1', delta: `"Hello, ` },
          { type: 'text-delta', id: 'text-1', delta: `world` },
          { type: 'text-delta', id: 'text-1', delta: `!"` },
          { type: 'text-delta', id: 'text-1', delta: ' }' },
          { type: 'text-end', id: 'text-1' },
          {
            type: 'finish',
            finishReason: 'stop',
            logprobs: undefined,
            usage: { inputTokens: 3, outputTokens: 10, totalTokens: 13 },
          },
        ],
      }),
    }),
  }),
  schema: z.object({ content: z.string() }),
  prompt: 'Hello, test!',
});
```

### Simulate UI Message Stream Responses

테스트/디버깅/데모 목적으로 [UI Message Stream](/docs/ai-sdk-ui/stream-protocol#ui-message-stream) 응답도 시뮬레이션할 수 있어.

다음은 Next 예시야:

```ts filename="route.ts"
import { simulateReadableStream } from 'ai';

export async function POST(req: Request) {
  return new Response(
    simulateReadableStream({
      initialDelayInMs: 1000, // Delay before the first chunk
      chunkDelayInMs: 300, // Delay between chunks
      chunks: [
        `data: {"type":"start","messageId":"msg-123"}\n\n`,
        `data: {"type":"text-start","id":"text-1"}\n\n`,
        `data: {"type":"text-delta","id":"text-1","delta":"This"}\n\n`,
        `data: {"type":"text-delta","id":"text-1","delta":" is an"}\n\n`,
        `data: {"type":"text-delta","id":"text-1","delta":" example."}\n\n`,
        `data: {"type":"text-end","id":"text-1"}\n\n`,
        `data: {"type":"finish"}\n\n`,
        `data: [DONE]\n\n`,
      ],
    }).pipeThrough(new TextEncoderStream()),
    {
      status: 200,
      headers: {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        Connection: 'keep-alive',
        'x-vercel-ai-ui-message-stream': 'v1',
      },
    },
  );
}
```
