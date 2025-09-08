# Settings (설정)

대규모 언어 모델(LLM)은 보통 출력을 조정할 수 있는 여러 설정을 제공해.

모델, [프롬프트](./prompts), 그리고 제공자별 추가 설정 외에, 모든 AI SDK 함수는 아래 공통 설정들을 지원해:

```ts highlight="3-5"
const result = await generateText({
  model: 'openai/gpt-4.1',
  maxOutputTokens: 512,
  temperature: 0.3,
  maxRetries: 5,
  prompt: 'Invent a new holiday and describe its traditions.',
});
```

<Note>
  일부 제공자는 모든 공통 설정을 지원하지 않아. 지원하지 않는 설정을 사용하면 경고가 발생해. 
  경고는 결과 객체의 `warnings` 프로퍼티에서 확인할 수 있어.
</Note>

### `maxOutputTokens`

생성할 최대 토큰 수.

### `temperature`

샘플링의 랜덤성(온도) 설정.

값은 제공자에 그대로 전달돼. 범위는 제공자/모델에 따라 달라. 대부분의 제공자에서 `0`은 거의 결정적이고, 값이 클수록 더 랜덤해져.

`temperature`와 `topP`는 둘 중 하나만 설정하는 걸 추천해. 둘 다 설정하진 마.

<Note>AI SDK 5.0부터 기본값은 더 이상 `0`이 아니야.</Note>

### `topP`

Nucleus 샘플링.

값은 제공자에 그대로 전달돼. 범위는 제공자/모델에 따라 달라. 보통 0~1 사이 수치야.
예: 0.1이면 누적 확률 질량 상위 10% 토큰만 고려한다는 뜻.

`temperature`와 `topP`는 둘 중 하나만 설정하는 걸 추천해.

### `topK`

다음 토큰 선택에서 상위 K개 옵션만 샘플링.

낮은 확률의 "롱테일" 응답을 제거하는 데 사용. 고급 사용 사례에만 추천. 보통은 `temperature`만으로 충분해.

### `presencePenalty`

프롬프트에 이미 있는 정보를 반복할 가능성에 대한 패널티.

값은 제공자에 그대로 전달돼. 범위는 제공자/모델에 따라 달라. 대부분의 제공자에서 `0`은 패널티 없음.

### `frequencyPenalty`

같은 단어/구절을 반복해서 사용할 가능성에 대한 패널티.

값은 제공자에 그대로 전달돼. 범위는 제공자/모델에 따라 달라. 대부분의 제공자에서 `0`은 패널티 없음.

### `stopSequences`

텍스트 생성을 중단시키는 시퀀스들.

설정되면, 모델은 지정된 시퀀스 중 하나가 생성되는 즉시 생성을 멈춰. 제공자별로 개수 제한이 있을 수 있어.

### `seed`

랜덤 샘플링에 사용할 시드(정수).
설정되고 모델이 지원하면, 호출 결과가 결정적이게 만들어줘.

### `maxRetries`

최대 재시도 횟수. 0으로 설정하면 재시도 비활성화. 기본값: `2`.

### `abortSignal`

호출을 취소(cancel)하는 데 사용할 수 있는 선택적 abort signal.

예: UI에서 전달받아 취소하거나, 타임아웃을 정의하는 데 사용.

#### 예: 타임아웃

```ts
const result = await generateText({
  model: openai('gpt-4o'),
  prompt: 'Invent a new holiday and describe its traditions.',
  abortSignal: AbortSignal.timeout(5000), // 5초
});
```

### `headers`

요청에 추가로 보낼 HTTP 헤더들. HTTP 기반 제공자에만 해당.

제공자가 지원하는 범위 내에서, 요청 헤더로 추가 정보를 전달할 수 있어. 예를 들어 어떤 옵저버빌리티 제공자는 `Prompt-Id` 같은 헤더를 지원해.

```ts
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

const result = await generateText({
  model: openai('gpt-4o'),
  prompt: 'Invent a new holiday and describe its traditions.',
  headers: {
    'Prompt-Id': 'my-prompt-id',
  },
});
```

<Note>
  `headers` 설정은 요청 단위 헤더용이야. 제공자 설정에도 `headers`를 넣을 수 있는데, 그 경우 해당 제공자가 보내는 모든 요청에 공통으로 적용돼.
</Note>
