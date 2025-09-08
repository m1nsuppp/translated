# Generating and Streaming Text (텍스트 생성과 스트리밍)

대규모 언어 모델(LLM)은 지시문과 정보를 담은 프롬프트에 대한 응답으로 텍스트를 생성할 수 있어.
예를 들어, 레시피 만들기, 이메일 초안 작성, 문서 요약 같은 일을 시킬 수 있지.

AI SDK Core는 LLM에서 텍스트를 생성하고 스트리밍하기 위한 두 가지 함수를 제공해:

- [`generateText`](#generatetext): 주어진 프롬프트와 모델로 텍스트를 생성해.
- [`streamText`](#streamtext): 주어진 프롬프트와 모델로 텍스트를 스트리밍해.

[툴 호출](./tools-and-tool-calling)이나 [구조화된 데이터 생성](./generating-structured-data) 같은 고급 기능도 텍스트 생성을 기반으로 구축돼 있어.

## `generateText`

[`generateText`](/docs/reference/ai-sdk-core/generate-text) 함수로 텍스트를 생성할 수 있어. 이 함수는 이메일 초안 작성, 웹페이지 요약 같은 비대화형 사용 사례나, 툴을 사용하는 에이전트에 적합해.

```tsx
import { generateText } from 'ai';

const { text } = await generateText({
  model: 'openai/gpt-4.1',
  prompt: 'Write a vegetarian lasagna recipe for 4 people.',
});
```

더 복잡한 지시와 내용을 담은 [고급 프롬프트](./prompts)도 사용할 수 있어:

```tsx
import { generateText } from 'ai';

const { text } = await generateText({
  model: 'openai/gpt-4.1',
  system: 'You are a professional writer. ' + 'You write simple, clear, and concise content.',
  prompt: `Summarize the following article in 3-5 sentences: ${article}`,
});
```

`generateText`의 결과 객체에는 필요한 데이터가 준비되었을 때 resolve되는 여러 프로미스가 있어:

- `result.content`: 마지막 스텝에서 생성된 콘텐츠.
- `result.text`: 생성된 텍스트.
- `result.reasoning`: 마지막 스텝에서 모델이 생성한 전체 추론.
- `result.reasoningText`: 모델의 추론 텍스트(일부 모델만 지원).
- `result.files`: 마지막 스텝에서 생성된 파일들.
- `result.sources`: 마지막 스텝에서 참고한 소스(일부 모델만 지원).
- `result.toolCalls`: 마지막 스텝에서 실행된 툴 호출들.
- `result.toolResults`: 마지막 스텝의 툴 실행 결과들.
- `result.finishReason`: 텍스트 생성을 종료한 이유.
- `result.usage`: 마지막 스텝에서의 사용량 정보.
- `result.totalUsage`: 모든 스텝을 합친 총 사용량(멀티-스텝인 경우).
- `result.warnings`: 모델 제공자의 경고(예: 지원되지 않는 설정).
- `result.request`: 추가 요청 정보.
- `result.response`: 응답 메시지/바디 등을 포함한 추가 응답 정보.
- `result.providerMetadata`: 제공자별 메타데이터.
- `result.steps`: 모든 스텝의 상세(중간 스텝 정보 확인용).
- `result.experimental_output`: `experimental_output` 스펙을 사용해 생성된 구조화 출력.

### 응답 헤더 & 바디 접근하기

가끔은 모델 제공자의 전체 응답에 접근해야 할 때가 있어(특정 헤더나 바디를 읽기 위해).

`response` 속성을 통해 원본 응답 헤더와 바디에 접근할 수 있어:

```ts
import { generateText } from 'ai';

const result = await generateText({
  // ...
});

console.log(JSON.stringify(result.response.headers, null, 2));
console.log(JSON.stringify(result.response.body, null, 2));
```

## `streamText`

모델과 프롬프트에 따라 LLM이 응답을 완성하는 데 최대 1분 정도 걸릴 수 있어. 챗봇이나 실시간 앱처럼 상호작용이 필요한 경우엔 이 지연이 치명적이지.

AI SDK Core는 텍스트 스트리밍을 단순화하는 [`streamText`](/docs/reference/ai-sdk-core/stream-text) 함수를 제공해:

```ts
import { streamText } from 'ai';

const result = streamText({
  model: 'openai/gpt-4.1',
  prompt: 'Invent a new holiday and describe its traditions.',
});

// example: use textStream as an async iterable
for await (const textPart of result.textStream) {
  console.log(textPart);
}
```

<Note>
  `result.textStream` is both a `ReadableStream` and an `AsyncIterable`.
</Note>

<Note type="warning">
  `streamText` immediately starts streaming and suppresses errors to prevent
  server crashes. Use the `onError` callback to log errors.
</Note>

`streamText`는 단독으로도 쓸 수 있고, [AI SDK UI](/examples/next-pages/basics/streaming-text-generation)나 [AI SDK RSC](/examples/next-app/basics/streaming-text-generation)와 함께 쓸 수도 있어.
통합을 쉽게 하라고 [AI SDK UI](/docs/ai-sdk-ui)와 연동되는 헬퍼도 같이 제공해:

- `result.toUIMessageStreamResponse()`: Next.js App Router API 라우트에서 쓸 수 있는 UI Message stream HTTP 응답을 생성(툴 호출 등 포함).
- `result.pipeUIMessageStreamToResponse()`: Node.js response-like 객체로 UI Message stream 델타를 씀.
- `result.toTextStreamResponse()`: 단순 텍스트 스트림 HTTP 응답을 생성.
- `result.pipeTextStreamToResponse()`: Node.js response-like 객체로 텍스트 델타를 씀.

<Note>
  `streamText`는 backpressure를 사용하고, 요청되는 만큼만 토큰을 생성해.
  스트림이 끝나려면 반드시 스트림을 소비해야 해.
</Note>

또 스트림이 끝나면 resolve되는 여러 프로미스를 제공해:

- `result.content`: 마지막 스텝에서 생성된 콘텐츠.
- `result.text`: 생성된 텍스트.
- `result.reasoning`: 모델이 생성한 전체 추론.
- `result.reasoningText`: 모델의 추론 텍스트(일부 모델만 지원).
- `result.files`: 마지막 스텝에서 모델이 생성한 파일들.
- `result.sources`: 마지막 스텝에서 참고한 소스(일부 모델만 지원).
- `result.toolCalls`: 마지막 스텝에서 실행된 툴 호출들.
- `result.toolResults`: 마지막 스텝에서 생성된 툴 결과들.
- `result.finishReason`: 텍스트 생성을 종료한 이유.
- `result.usage`: 마지막 스텝에서의 사용량 정보.
- `result.totalUsage`: 모든 스텝을 합친 총 사용량(멀티-스텝인 경우).
- `result.warnings`: 모델 제공자의 경고(예: 지원되지 않는 설정).
- `result.steps`: 모든 스텝의 상세(중간 스텝 정보 확인용).
- `result.request`: 마지막 스텝의 추가 요청 정보.
- `result.response`: 마지막 스텝의 추가 응답 정보.
- `result.providerMetadata`: 마지막 스텝의 제공자별 메타데이터.

### `onError` 콜백

`streamText`는 즉시 스트리밍을 시작해서 대기 없이 데이터를 보낼 수 있게 해. 에러는 서버 크래시를 막기 위해 throw되지 않고 스트림의 일부가 돼.

에러 로깅이 필요하면 에러 발생 시 호출되는 `onError` 콜백을 제공하면 돼.

```tsx highlight="6-8"
import { streamText } from 'ai';

const result = streamText({
  model: 'openai/gpt-4.1',
  prompt: 'Invent a new holiday and describe its traditions.',
  onError({ error }) {
    console.error(error); // your error logging logic here
  },
});
```

### `onChunk` 콜백

`streamText`를 사용할 때, 스트림의 각 청크마다 호출되는 `onChunk` 콜백을 제공할 수 있어.

다음 타입의 청크를 받게 돼:

- `text`
- `reasoning`
- `source`
- `tool-call`
- `tool-input-start`
- `tool-input-delta`
- `tool-result`
- `raw`

```tsx highlight="6-11"
import { streamText } from 'ai';

const result = streamText({
  model: 'openai/gpt-4.1',
  prompt: 'Invent a new holiday and describe its traditions.',
  onChunk({ chunk }) {
    // implement your own logic here, e.g.:
    if (chunk.type === 'text') {
      console.log(chunk.text);
    }
  },
});
```

### `onFinish` 콜백

`streamText`는 스트림이 끝났을 때 호출되는 `onFinish` 콜백을 제공해(
[API Reference](/docs/reference/ai-sdk-core/stream-text#on-finish)
).
텍스트, 사용량, 종료 이유, 메시지, 스텝, 총 사용량 등의 정보를 받을 수 있어:

```tsx highlight="6-8"
import { streamText } from 'ai';

const result = streamText({
  model: 'openai/gpt-4.1',
  prompt: 'Invent a new holiday and describe its traditions.',
  onFinish({ text, finishReason, usage, response, steps, totalUsage }) {
    // your own logic, e.g. for saving the chat history or recording usage

    const messages = response.messages; // messages that were generated
  },
});
```

### `fullStream` 프로퍼티

모든 이벤트가 포함된 스트림을 `fullStream` 프로퍼티로 읽을 수 있어.
직접 UI를 구현하거나 스트림을 다르게 처리하고 싶을 때 유용해.
사용 예시는 아래와 같아:

```tsx
import { streamText } from 'ai';
import { z } from 'zod';

const result = streamText({
  model: 'openai/gpt-4.1',
  tools: {
    cityAttractions: {
      inputSchema: z.object({ city: z.string() }),
      execute: async ({ city }) => ({
        attractions: ['attraction1', 'attraction2', 'attraction3'],
      }),
    },
  },
  prompt: 'What are some San Francisco tourist attractions?',
});

for await (const part of result.fullStream) {
  switch (part.type) {
    case 'start': {
      // handle start of stream
      break;
    }
    case 'start-step': {
      // handle start of step
      break;
    }
    case 'text-start': {
      // handle text start
      break;
    }
    case 'text-delta': {
      // handle text delta here
      break;
    }
    case 'text-end': {
      // handle text end
      break;
    }
    case 'reasoning-start': {
      // handle reasoning start
      break;
    }
    case 'reasoning-delta': {
      // handle reasoning delta here
      break;
    }
    case 'reasoning-end': {
      // handle reasoning end
      break;
    }
    case 'source': {
      // handle source here
      break;
    }
    case 'file': {
      // handle file here
      break;
    }
    case 'tool-call': {
      switch (part.toolName) {
        case 'cityAttractions': {
          // handle tool call here
          break;
        }
      }
      break;
    }
    case 'tool-input-start': {
      // handle tool input start
      break;
    }
    case 'tool-input-delta': {
      // handle tool input delta
      break;
    }
    case 'tool-input-end': {
      // handle tool input end
      break;
    }
    case 'tool-result': {
      switch (part.toolName) {
        case 'cityAttractions': {
          // handle tool result here
          break;
        }
      }
      break;
    }
    case 'tool-error': {
      // handle tool error
      break;
    }
    case 'finish-step': {
      // handle finish step
      break;
    }
    case 'finish': {
      // handle finish here
      break;
    }
    case 'error': {
      // handle error here
      break;
    }
    case 'raw': {
      // handle raw value
      break;
    }
  }
}
```

### 스트림 변환

`experimental_transform` 옵션을 사용하면 스트림을 변환할 수 있어.
예를 들어 텍스트 스트림을 필터링/변경/스무딩하는 데 유용해.

변환은 콜백 호출 및 프로미스 resolve 전에 적용돼. 예를 들어 모든 텍스트를 대문자로 바꾸는 변환을 사용하면, `onFinish` 콜백은 변환된 텍스트를 받게 돼.

#### 스트림 스무딩

AI SDK Core는 텍스트 스트리밍을 스무딩하는 데 쓸 수 있는 [`smoothStream` 함수](/docs/reference/ai-sdk-core/smooth-stream)를 제공해.

```tsx highlight="6"
import { smoothStream, streamText } from 'ai';

const result = streamText({
  model,
  prompt,
  experimental_transform: smoothStream(),
});
```

#### 커스텀 변환

커스텀 변환도 구현할 수 있어. 변환 함수는 모델에 제공되는 툴들을 인수로 받고, 스트림을 변환하는 함수를 반환해.
툴은 제네릭일 수도 있고, 네가 실제로 사용하는 툴로 제한할 수도 있어.

아래는 모든 텍스트를 대문자로 바꾸는 커스텀 변환 예시야:

```ts
const upperCaseTransform =
  <TOOLS extends ToolSet>() =>
  (options: { tools: TOOLS; stopStream: () => void }) =>
    new TransformStream<TextStreamPart<TOOLS>, TextStreamPart<TOOLS>>({
      transform(chunk, controller) {
        controller.enqueue(
          // for text chunks, convert the text to uppercase:
          chunk.type === 'text' ? { ...chunk, text: chunk.text.toUpperCase() } : chunk,
        );
      },
    });
```

`stopStream` 함수로 스트림을 중단할 수도 있어. 예를 들어 모델 가드레일을 위반하는 부적절한 콘텐츠가 생성될 때 스트림을 멈추는 데 유용해.

`stopStream`을 호출할 때는, 올바른 스트림 형태를 보장하고 모든 콜백이 호출되도록 `step-finish`와 `finish` 이벤트를 시뮬레이션하는 게 중요해.

```ts
const stopWordTransform =
  <TOOLS extends ToolSet>() =>
  ({ stopStream }: { stopStream: () => void }) =>
    new TransformStream<TextStreamPart<TOOLS>, TextStreamPart<TOOLS>>({
      // note: this is a simplified transformation for testing;
      // in a real-world version more there would need to be
      // stream buffering and scanning to correctly emit prior text
      // and to detect all STOP occurrences.
      transform(chunk, controller) {
        if (chunk.type !== 'text') {
          controller.enqueue(chunk);
          return;
        }

        if (chunk.text.includes('STOP')) {
          // stop the stream
          stopStream();

          // simulate the finish-step event
          controller.enqueue({
            type: 'finish-step',
            finishReason: 'stop',
            logprobs: undefined,
            usage: {
              completionTokens: NaN,
              promptTokens: NaN,
              totalTokens: NaN,
            },
            request: {},
            response: {
              id: 'response-id',
              modelId: 'mock-model-id',
              timestamp: new Date(0),
            },
            warnings: [],
            isContinued: false,
          });

          // simulate the finish event
          controller.enqueue({
            type: 'finish',
            finishReason: 'stop',
            logprobs: undefined,
            usage: {
              completionTokens: NaN,
              promptTokens: NaN,
              totalTokens: NaN,
            },
            response: {
              id: 'response-id',
              modelId: 'mock-model-id',
              timestamp: new Date(0),
            },
          });

          return;
        }

        controller.enqueue(chunk);
      },
    });
```

#### 여러 변환 적용하기

여러 개의 변환을 배열로 제공할 수도 있어. 제공된 순서대로 적용돼.

```tsx highlight="4"
const result = streamText({
  model,
  prompt,
  experimental_transform: [firstTransform, secondTransform],
});
```

## Sources

[Perplexity](/providers/ai-sdk-providers/perplexity#sources)나
[Google Generative AI](/providers/ai-sdk-providers/google-generative-ai#sources) 같은 일부 제공자는 응답에 소스를 포함해.

현재 소스는 응답을 근거지우는 웹페이지로 제한돼 있어. `sources` 프로퍼티를 통해 접근할 수 있어.

각 `url` 소스는 다음 프로퍼티를 포함해:

- `id`: 소스의 ID.
- `url`: 소스의 URL.
- `title`: 소스의 선택적 제목.
- `providerMetadata`: 제공자 메타데이터.

`generateText`를 사용할 때는 `sources` 프로퍼티로 소스에 접근할 수 있어:

```ts
const result = await generateText({
  model: google('gemini-2.5-flash'),
  tools: {
    google_search: google.tools.googleSearch({}),
  },
  prompt: 'List the top 5 San Francisco news from the past week.',
});

for (const source of result.sources) {
  if (source.sourceType === 'url') {
    console.log('ID:', source.id);
    console.log('Title:', source.title);
    console.log('URL:', source.url);
    console.log('Provider metadata:', source.providerMetadata);
    console.log();
  }
}
```

`streamText`를 사용할 때는 `fullStream` 프로퍼티로 소스에 접근할 수 있어:

```tsx
const result = streamText({
  model: google('gemini-2.5-flash'),
  tools: {
    google_search: google.tools.googleSearch({}),
  },
  prompt: 'List the top 5 San Francisco news from the past week.',
});

for await (const part of result.fullStream) {
  if (part.type === 'source' && part.sourceType === 'url') {
    console.log('ID:', part.id);
    console.log('Title:', part.title);
    console.log('URL:', part.url);
    console.log('Provider metadata:', part.providerMetadata);
    console.log();
  }
}
```

소스는 `result.sources` 프로미스에서도 사용할 수 있어.
