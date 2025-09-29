# 이미지 생성

> 경고: 이미지 생성은 실험적 기능이야.

AI SDK는 이미지 모델로 프롬프트를 기반으로 이미지를 생성하는 `generateImage` 함수를 제공해.

```tsx
import { experimental_generateImage as generateImage } from 'ai';
import { openai } from '@ai-sdk/openai';

const { image } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt: 'Santa Claus driving a Cadillac',
});
```

생성된 이미지 데이터는 `base64` 또는 `uint8Array` 속성으로 접근할 수 있어:

```tsx
const base64 = image.base64; // base64 이미지 데이터
const uint8Array = image.uint8Array; // Uint8Array 이진 데이터
```

## 설정

### 크기와 화면비

모델에 따라 크기(size) 또는 화면비(aspect ratio)를 지정할 수 있어.

##### 크기(Size)

크기는 `{width}x{height}` 형태의 문자열로 지정해.
각 모델/프로바이더가 지원하는 크기만 사용할 수 있고, 지원 범위는 모델마다 달라.

```tsx highlight={"7"}
import { experimental_generateImage as generateImage } from 'ai';
import { openai } from '@ai-sdk/openai';

const { image } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt: 'Santa Claus driving a Cadillac',
  size: '1024x1024',
});
```

##### 화면비(Aspect Ratio)

화면비는 `{width}:{height}` 형태의 문자열로 지정해.
각 모델/프로바이더가 지원하는 화면비만 사용할 수 있고, 지원 범위는 모델마다 달라.

```tsx highlight={"7"}
import { experimental_generateImage as generateImage } from 'ai';
import { vertex } from '@ai-sdk/google-vertex';

const { image } = await generateImage({
  model: vertex.image('imagen-3.0-generate-002'),
  prompt: 'Santa Claus driving a Cadillac',
  aspectRatio: '16:9',
});
```

### 여러 장 이미지 생성

`generateImage`는 한 번에 여러 장 이미지를 생성하는 것도 지원해:

```tsx highlight={"7"}
import { experimental_generateImage as generateImage } from 'ai';
import { openai } from '@ai-sdk/openai';

const { images } = await generateImage({
  model: openai.image('dall-e-2'),
  prompt: 'Santa Claus driving a Cadillac',
  n: 4, // 생성할 이미지 개수
});
```

> 참고: 요청한 이미지 수를 생성하기 위해 필요하면 모델을 여러 번(병렬로) 호출해.

각 이미지 모델에는 한 번의 API 호출로 생성할 수 있는 이미지 수의 내부 제한이 있어. SDK는 `n` 파라미터를 사용할 때 이 제한을 고려해 자동으로 배치/분할 호출을 해줘. (예: DALL-E 3는 호출당 1장, DALL-E 2는 최대 10장)

필요하면 `maxImagesPerCall` 설정으로 이 동작을 오버라이드할 수 있어. 특히 새/커스텀 모델을 사용할 때 기본 배치 크기가 최적이 아닐 수 있으면 유용해:

```tsx
const { images } = await generateImage({
  model: openai.image('dall-e-2'),
  prompt: 'Santa Claus driving a Cadillac',
  maxImagesPerCall: 5, // 기본 배치 크기 덮어쓰기
  n: 10, // 5장씩 2번 호출
});
```

### 시드(Seed) 지정

`generateImage`에 시드를 지정해 생성 결과를 제어할 수 있어.
모델이 지원한다면 같은 시드는 항상 같은 이미지를 만들어.

```tsx highlight={"7"}
import { experimental_generateImage as generateImage } from 'ai';
import { openai } from '@ai-sdk/openai';

const { image } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt: 'Santa Claus driving a Cadillac',
  seed: 1234567890,
});
```

### 프로바이더별 설정

이미지 모델은 프로바이더/모델별 특화된 설정을 가지는 경우가 많아.
이런 옵션은 `providerOptions` 파라미터로 `generateImage`에 전달할 수 있어.
아래 예시에서 프로바이더(`openai`) 키 아래의 옵션은 요청 바디 속성으로 들어가.

```tsx highlight={"9"}
import { experimental_generateImage as generateImage } from 'ai';
import { openai } from '@ai-sdk/openai';

const { image } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt: 'Santa Claus driving a Cadillac',
  size: '1024x1024',
  providerOptions: {
    openai: { style: 'vivid', quality: 'hd' },
  },
});
```

### 중단 신호와 타임아웃

`generateImage`는 선택적 `abortSignal`(타입: `AbortSignal`)을 받아서
이미지 생성을 중단하거나 타임아웃을 설정할 수 있어.

```ts highlight={"7"}
import { openai } from '@ai-sdk/openai';
import { experimental_generateImage as generateImage } from 'ai';

const { image } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt: 'Santa Claus driving a Cadillac',
  abortSignal: AbortSignal.timeout(1000), // 1초 후 중단
});
```

### 커스텀 헤더

`generateImage`는 선택적 `headers`(타입: `Record<string, string>`)를 받아
이미지 생성 요청에 커스텀 헤더를 추가할 수 있어.

```ts highlight={"7"}
import { openai } from '@ai-sdk/openai';
import { experimental_generateImage as generateImage } from 'ai';

const { image } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt: 'Santa Claus driving a Cadillac',
  headers: { 'X-Custom-Header': 'custom-value' },
});
```

### 경고(Warnings)

모델이 경고를 반환하는 경우(예: 미지원 파라미터), 응답의 `warnings` 속성에서 확인할 수 있어.

```tsx
const { image, warnings } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt: 'Santa Claus driving a Cadillac',
});
```

### 추가 프로바이더 메타데이터

일부 프로바이더는 결과 전체 또는 이미지별로 추가 메타데이터를 노출해.

```tsx
const prompt = 'Santa Claus driving a Cadillac';

const { image, providerMetadata } = await generateImage({
  model: openai.image('dall-e-3'),
  prompt,
});

const revisedPrompt = providerMetadata.openai.images[0]?.revisedPrompt;

console.log({
  prompt,
  revisedPrompt,
});
```

반환된 `providerMetadata`의 최상위 키는 프로바이더 이름이야. 내부 값은 메타데이터고,
`images` 키가 항상 존재하며 최상위 `images` 길이와 동일한 길이의 배열이야.

---

한국어 번역본을 작성했어. 문서 위치: `image-generation.ko.md`

### 에러 처리

`generateImage`가 유효한 이미지를 생성하지 못하면 [`AI_NoImageGeneratedError`](/docs/reference/ai-sdk-errors/ai-no-image-generated-error)를 던져.

이 에러는 AI 프로바이더가 이미지 생성에 실패할 때 발생해. 가능한 원인은:

- 모델이 응답 생성에 실패함
- 모델이 생성한 응답을 파싱할 수 없음

로그에 도움이 되도록 다음 정보를 보존해:

- `responses`: 이미지 모델 응답 메타데이터(타임스탬프, 모델, 헤더 등)
- `cause`: 에러의 실제 원인(더 자세한 에러 처리에 활용 가능)

```ts
import { generateImage, NoImageGeneratedError } from 'ai';

try {
  await generateImage({ model, prompt });
} catch (error) {
  if (NoImageGeneratedError.isInstance(error)) {
    console.log('NoImageGeneratedError');
    console.log('Cause:', error.cause);
    console.log('Responses:', error.responses);
  }
}
```

## 언어 모델로 이미지 생성

일부 언어 모델(예: Google `gemini-2.5-flash-image-preview`)은 이미지가 포함된 멀티모달 출력을 지원해.
이런 모델에서는 응답의 `files` 속성으로 생성된 이미지를 접근할 수 있어.

```ts
import { google } from '@ai-sdk/google';
import { generateText } from 'ai';

const result = await generateText({
  model: google('gemini-2.5-flash-image-preview'),
  prompt: 'Generate an image of a comic cat',
});

for (const file of result.files) {
  if (file.mediaType.startsWith('image/')) {
    // file 객체는 여러 데이터 형식을 제공해:
    // base64 문자열, Uint8Array 이진 데이터, 타입 확인 등
    // - file.base64: string (data URL 형식)
    // - file.uint8Array: Uint8Array (이진 데이터)
    // - file.mediaType: string (예: "image/png")
  }
}
```
