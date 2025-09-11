# 프롬프트 (Prompts)

프롬프트란 LLM(대규모 언어 모델)에게 수행할 작업을 지시하기 위한 입력임. 길 묻듯이 질문을 명확히 할수록 결과 품질이 좋아짐.

여러 LLM 제공자는 프롬프트를 구성하기 위한 다양한 인터페이스(역할/메시지 타입 등)를 제공함. 강력하지만 학습곡선이 있음. 단순화를 위해 AI SDK는 텍스트, 메시지, 시스템 프롬프트를 지원함.

## 텍스트 프롬프트 (Text Prompts)

텍스트 프롬프트는 문자열임. 반복 생성 등 단순 유즈케이스에 적합함. 템플릿 리터럴로 변수를 주입해 동적 구성 가능함.

```ts highlight="3"
const result = await generateText({
  model: 'openai/gpt-4.1',
  prompt: 'Invent a new holiday and describe its traditions.',
});
```

동적 데이터 주입 예시:

```ts highlight="3-5"
const result = await generateText({
  model: 'openai/gpt-4.1',
  prompt:
    `I am planning a trip to ${destination} for ${lengthOfStay} days. ` +
    `Please suggest the best tourist activities for me to do.`,
});
```

## 시스템 프롬프트 (System Prompts)

시스템 프롬프트는 모델의 거동과 응답을 안내/제약하는 초기 지시문임. `system` 프로퍼티로 설정함. `prompt`/`messages`와 함께 동작함.

```ts highlight="3-6"
const result = await generateText({
  model: 'openai/gpt-4.1',
  system:
    `You help planning travel itineraries. ` +
    `Respond to the users' request with a list ` +
    `of the best stops to make in their destination.`,
  prompt:
    `I am planning a trip to ${destination} for ${lengthOfStay} days. ` +
    `Please suggest the best tourist activities for me to do.`,
});
```

<Note>
  메시지 프롬프트를 사용할 때는 시스템 프롬프트 대신 시스템 메시지를 쓸 수도 있음.
</Note>

## 메시지 프롬프트 (Message Prompts)

메시지 프롬프트는 user/assistant/tool 메시지 배열임. 채팅 인터페이스 및 멀티모달 프롬프트에 적합함. `messages` 프로퍼티로 설정함.

각 메시지는 `role`과 `content`를 가짐. content는 텍스트(유저/어시스턴트) 또는 해당 타입에 맞는 파트 배열일 수 있음.

```ts highlight="3-7"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    { role: 'user', content: 'Hi!' },
    { role: 'assistant', content: 'Hello, how can I help?' },
    { role: 'user', content: 'Where can I buy the best Currywurst in Berlin?' },
  ],
});
```

`content`에 문자열 대신, 텍스트와 기타 파트를 섞은 배열을 보낼 수도 있음.

<Note type="warning">
  모든 모델이 모든 메시지/콘텐츠 타입을 지원하지 않음. 예: 일부 모델은 멀티모달 입력 또는 tool 메시지를 처리 못함. [지원 모델 역량 참고](./providers-and-models#model-capabilities).
</Note>

### Provider Options

제공사별 기능을 활성화하기 위한 메타데이터를 3 레벨에서 전달 가능함.

#### 함수 호출 레벨 (Function Call Level)

[`streamText`](/docs/reference/ai-sdk-core/stream-text#provider-options) 또는 [`generateText`](/docs/reference/ai-sdk-core/generate-text#provider-options)는 `providerOptions`를 받음. 특정 위치 제어가 필요 없을 때 이 레벨에서 설정함.

```ts
const { text } = await generateText({
  model: azure('your-deployment-name'),
  providerOptions: {
    openai: {
      reasoningEffort: 'low',
    },
  },
});
```

#### 메시지 레벨 (Message Level)

메시지 단위로 세밀 제어 필요 시, 메시지 오브젝트에 `providerOptions` 전달함.

```ts
import { ModelMessage } from 'ai';

const messages: ModelMessage[] = [
  {
    role: 'system',
    content: 'Cached system message',
    providerOptions: {
      // 시스템 메시지에 캐시 컨트롤 브레이크포인트 설정
      anthropic: { cacheControl: { type: 'ephemeral' } },
    },
  },
];
```

#### 메시지 파트 레벨 (Message Part Level)

일부 제공사 옵션은 파트 레벨에서만 설정 가능함.

```ts
import { ModelMessage } from 'ai';

const messages: ModelMessage[] = [
  {
    role: 'user',
    content: [
      {
        type: 'text',
        text: 'Describe the image in detail.',
        providerOptions: {
          openai: { imageDetail: 'low' },
        },
      },
      {
        type: 'image',
        image:
          'https://github.com/vercel/ai/blob/main/examples/ai-core/data/comic-cat.png?raw=true',
        // 이미지 파트에 대한 세부 설정
        providerOptions: {
          openai: { imageDetail: 'low' },
        },
      },
    ],
  },
];
```

<Note type="warning">
  AI SDK UI 훅인 [`useChat`](/docs/reference/ai-sdk-ui/use-chat)은 `UIMessage` 배열을 반환하며 provider options 미지원임. `providerOptions`가 필요한 메시지(또는 파트)를 적용/추가하기 전, [`convertToModelMessages`](/docs/reference/ai-sdk-ui/convert-to-core-messages)로 `UIMessage`를 [`ModelMessage`](/docs/reference/ai-sdk-core/model-message)로 변환 권장함.
</Note>

### User Messages

#### Text Parts

텍스트는 가장 일반적 콘텐츠 타입임(문자열). 텍스트만 보내면 `content`에 문자열로 충분하지만, 여러 파트를 배열로 섞어 보낼 수도 있음.

```ts highlight="7-10"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'text',
          text: 'Where can I buy the best Currywurst in Berlin?',
        },
      ],
    },
  ],
});
```

#### Image Parts

유저 메시지에 이미지 파트를 포함할 수 있음. 이미지 타입:

- base64 이미지:
  - base-64 인코딩 문자열
  - data URL 문자열 (예: `data:image/png;base64,...`)
- 바이너리 이미지:
  - `ArrayBuffer`
  - `Uint8Array`
  - `Buffer`
- URL:
  - http(s) URL 문자열 (예: `https://example.com/image.png`)
  - `URL` 객체 (예: `new URL('https://example.com/image.png')`)

##### 예: 바이너리 이미지(Buffer)

```ts highlight="8-11"
const result = await generateText({
  model,
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'Describe the image in detail.' },
        {
          type: 'image',
          image: fs.readFileSync('./data/comic-cat.png'),
        },
      ],
    },
  ],
});
```

##### 예: Base-64 인코딩 이미지(string)

```ts highlight="8-11"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'Describe the image in detail.' },
        {
          type: 'image',
          image: fs.readFileSync('./data/comic-cat.png').toString('base64'),
        },
      ],
    },
  ],
});
```

##### 예: 이미지 URL(string)

```ts highlight="8-12"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'Describe the image in detail.' },
        {
          type: 'image',
          image:
            'https://github.com/vercel/ai/blob/main/examples/ai-core/data/comic-cat.png?raw=true',
        },
      ],
    },
  ],
});
```

#### File Parts

<Note type="warning">
  파일 파트를 지원하는 제공사/모델은 제한적임: [Google Generative AI](/providers/ai-sdk-providers/google-generative-ai), [Google Vertex AI](/providers/ai-sdk-providers/google-vertex), [OpenAI](/providers/ai-sdk-providers/openai) (`gpt-4o-audio-preview`의 `wav`/`mp3`), [Anthropic](/providers/ai-sdk-providers/anthropic), [OpenAI](/providers/ai-sdk-providers/openai) (`pdf`).
</Note>

유저 메시지에 파일 파트를 포함할 수 있음. 파일 타입:

- base64 파일:
  - base-64 인코딩 문자열
  - data URL 문자열 (예: `data:image/png;base64,...`)
- 바이너리 데이터:
  - `ArrayBuffer`
  - `Uint8Array`
  - `Buffer`
- URL:
  - http(s) URL 문자열 (예: `https://example.com/some.pdf`)
  - `URL` 객체 (예: `new URL('https://example.com/some.pdf')`)

전송 파일의 MIME 타입 명시 필요함.

##### 예: Buffer에서 PDF 전송

```ts highlight="12-15"
import { google } from '@ai-sdk/google';
import { generateText } from 'ai';

const result = await generateText({
  model: google('gemini-1.5-flash'),
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'What is the file about?' },
        {
          type: 'file',
          mediaType: 'application/pdf',
          data: fs.readFileSync('./data/example.pdf'),
          filename: 'example.pdf', // optional, 공급사에 따라 미사용
        },
      ],
    },
  ],
});
```

##### 예: Buffer에서 mp3 전송

```ts highlight="12-14"
import { openai } from '@ai-sdk/openai';
import { generateText } from 'ai';

const result = await generateText({
  model: openai('gpt-4o-audio-preview'),
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'What is the audio saying?' },
        {
          type: 'file',
          mediaType: 'audio/mpeg',
          data: fs.readFileSync('./data/galileo.mp3'),
        },
      ],
    },
  ],
});
```

#### 커스텀 다운로드 함수 (실험적)

스로틀링/재시도/인증/캐시 등을 구현하기 위해 커스텀 다운로드 함수를 사용할 수 있음. 기본 구현은 모델이 직접 지원하지 않는 파일을 자동 병렬 다운로드함. `experimental_download`로 전달함.

```ts
const result = await generateText({
  model: openai('gpt-4o'),
  experimental_download: async (
    requestedDownloads: Array<{
      url: URL;
      isUrlSupportedByModel: boolean;
    }>,
  ): PromiseLike<
    Array<{
      data: Uint8Array;
      mediaType: string | undefined;
    } | null>
  > => {
    // ... download the files and return an array with similar order
  },
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'file',
          data: new URL('https://api.company.com/private/document.pdf'),
          mediaType: 'application/pdf',
        },
      ],
    },
  ],
});
```

<Note>
  `experimental_download` 옵션은 실험적 기능이며, 향후 변경될 수 있음.
</Note>

### Assistant Messages

`assistant` 역할의 메시지임. 보통 이전 어시스턴트 응답을 담고, 텍스트/추론/도구 호출 파트를 포함할 수 있음.

#### 예: 텍스트 콘텐츠만 포함한 어시스턴트 메시지

```ts highlight="5"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    { role: 'user', content: 'Hi!' },
    { role: 'assistant', content: 'Hello, how can I help?' },
  ],
});
```

#### 예: 텍스트 콘텐츠를 배열 파트로 포함

```ts highlight="7"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    { role: 'user', content: 'Hi!' },
    {
      role: 'assistant',
      content: [{ type: 'text', text: 'Hello, how can I help?' }],
    },
  ],
});
```

#### 예: 도구 호출 파트를 포함한 어시스턴트 메시지

```ts highlight="7-14"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    { role: 'user', content: 'How many calories are in this block of cheese?' },
    {
      role: 'assistant',
      content: [
        {
          type: 'tool-call',
          toolCallId: '12345',
          toolName: 'get-nutrition-data',
          input: { cheese: 'Roquefort' },
        },
      ],
    },
  ],
});
```

#### 예: 파일 파트를 포함한 어시스턴트 메시지

<Note>
  모델이 생성한 파일에 대한 파트임. 지원 모델과 파일 형식 제한적임.
</Note>

```ts highlight="9-11"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    { role: 'user', content: 'Generate an image of a roquefort cheese!' },
    {
      role: 'assistant',
      content: [
        {
          type: 'file',
          mediaType: 'image/png',
          data: fs.readFileSync('./data/roquefort.jpg'),
        },
      ],
    },
  ],
});
```

### Tool messages

<Note>
  [Tools](/docs/foundations/tools) (function calling) 는 LLM 기능을 확장하는 프로그램임. 외부 API 호출부터 UI 내부 함수 호출까지 가능. 자세한 내용은 [다음 섹션](/docs/foundations/tools) 참조.
</Note>

도구 호출 지원 모델의 경우, 어시스턴트 메시지는 도구 호출 파트를, 도구 메시지는 도구 출력 파트를 포함할 수 있음. 단일 어시스턴트 메시지가 여러 도구를 병렬 호출할 수 있고, 단일 도구 메시지가 여러 결과를 포함할 수 있음.

```ts highlight="14-42"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'text',
          text: 'How many calories are in this block of cheese?',
        },
        { type: 'image', image: fs.readFileSync('./data/roquefort.jpg') },
      ],
    },
    {
      role: 'assistant',
      content: [
        {
          type: 'tool-call',
          toolCallId: '12345',
          toolName: 'get-nutrition-data',
          input: { cheese: 'Roquefort' },
        },
        // 병렬로 더 많은 도구 호출 가능
      ],
    },
    {
      role: 'tool',
      content: [
        {
          type: 'tool-result',
          toolCallId: '12345', // 위 호출 ID와 일치해야 함
          toolName: 'get-nutrition-data',
          output: {
            type: 'json',
            value: {
              name: 'Cheese, roquefort',
              calories: 369,
              fat: 31,
              protein: 22,
            },
          },
        },
        // 병렬로 더 많은 도구 결과 가능
      ],
    },
  ],
});
```

#### 멀티모달 도구 결과 (Multi-modal Tool Results)

<Note type="warning">
  멀티파트 도구 결과는 실험적이며 Anthropic만 지원함.
</Note>

도구 결과는 텍스트+이미지 등 다중 파트/모달 가능함. 파트의 `experimental_content`를 사용해 지정 가능함.

```ts highlight="24-46"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    // ...
    {
      role: 'tool',
      content: [
        {
          type: 'tool-result',
          toolCallId: '12345', // 위 호출 ID와 일치해야 함
          toolName: 'get-nutrition-data',
          // 멀티파트 미지원 모델 대비 일반 출력 포함
          output: {
            type: 'json',
            value: {
              name: 'Cheese, roquefort',
              calories: 369,
              fat: 31,
              protein: 22,
            },
          },
        },
        {
          type: 'tool-result',
          toolCallId: '12345', // 위 호출 ID와 일치해야 함
          toolName: 'get-nutrition-data',
          // 멀티파트 지원 모델 대비 멀티 콘텐츠 파트 포함
          output: {
            type: 'content',
            value: [
              {
                type: 'text',
                text: 'Here is an image of the nutrition data for the cheese:',
              },
              {
                type: 'media',
                data: fs
                  .readFileSync('./data/roquefort-nutrition-data.png')
                  .toString('base64'),
                mediaType: 'image/png',
              },
            ],
          },
        },
      ],
    },
  ],
});
```

### 시스템 메시지 (System Messages)

시스템 메시지는 유저 메시지 이전에 보내 모델의 거동을 안내함. 대안으로 `system` 프로퍼티 사용 가능함.

```ts highlight="4"
const result = await generateText({
  model: 'openai/gpt-4.1',
  messages: [
    { role: 'system', content: 'You help planning travel itineraries.' },
    {
      role: 'user',
      content:
        'I am planning a trip to Berlin for 3 days. Please suggest the best tourist activities for me to do.',
    },
  ],
});
```
