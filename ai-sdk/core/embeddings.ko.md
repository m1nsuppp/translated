# 임베딩(Embeddings)

임베딩은 단어, 구절, 이미지 등을 고차원 공간의 벡터로 표현하는 방법이야.
이 공간에서 유사한 단어들은 서로 가깝게 위치하고, 두 벡터 사이의 거리는 유사도를 측정하는 데 사용돼.

## 단일 값 임베딩하기

AI SDK는 단일 값을 임베딩하는 [`embed`](/docs/reference/ai-sdk-core/embed) 함수를 제공해.
이는 유사 단어/구절 찾기나 텍스트 클러스터링 같은 작업에 유용해.
임베딩 모델로 예를 들면 `openai.textEmbeddingModel('text-embedding-3-large')`나 `mistral.textEmbeddingModel('mistral-embed')`를 사용할 수 있어.

```tsx
import { embed } from 'ai';
import { openai } from '@ai-sdk/openai';

// 'embedding'은 단일 임베딩 벡터(number[])야
const { embedding } = await embed({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  value: 'sunny day at the beach',
});
```

## 여러 값 임베딩하기

데이터 적재(예: RAG용 데이터 스토어 준비) 시에는 여러 값을 한 번에 임베딩(배치 임베딩)하는 게 유용해.

AI SDK는 이를 위한 [`embedMany`](/docs/reference/ai-sdk-core/embed-many) 함수를 제공해.
`embed`와 유사하게 임베딩 모델을 지정해 사용할 수 있어
(예: `openai.textEmbeddingModel('text-embedding-3-large')`, `mistral.textEmbeddingModel('mistral-embed')`).

```tsx
import { openai } from '@ai-sdk/openai';
import { embedMany } from 'ai';

// 'embeddings'는 임베딩 배열(number[][])이야. 입력 순서와 동일하게 정렬돼.
const { embeddings } = await embedMany({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  values: ['sunny day at the beach', 'rainy afternoon in the city', 'snowy night in the mountains'],
});
```

## 임베딩 유사도

임베딩한 값들 간 유사도는 [`cosineSimilarity`](/docs/reference/ai-sdk-core/cosine-similarity)로 계산할 수 있어.
데이터셋에서 비슷한 단어나 구를 찾거나, 유사도 기반 랭킹/필터링을 할 때 유용해.

```ts highlight={"2,10"}
import { openai } from '@ai-sdk/openai';
import { cosineSimilarity, embedMany } from 'ai';

const { embeddings } = await embedMany({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  values: ['sunny day at the beach', 'rainy afternoon in the city'],
});

console.log(`cosine similarity: ${cosineSimilarity(embeddings[0], embeddings[1])}`);
```

## 토큰 사용량

여러 제공자는 임베딩에 사용된 토큰 수로 과금해.
`embed`와 `embedMany`는 결과 객체의 `usage`에 토큰 사용량 정보를 포함해.

```ts highlight={"4,9"}
import { openai } from '@ai-sdk/openai';
import { embed } from 'ai';

const { embedding, usage } = await embed({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  value: 'sunny day at the beach',
});

console.log(usage); // { tokens: 10 }
```

## 설정(Settings)

### Provider Options

임베딩 모델 설정은 `providerOptions`로 제공자별 파라미터를 조정할 수 있어.

```ts highlight={"5-9"}
import { openai } from '@ai-sdk/openai';
import { embed } from 'ai';

const { embedding } = await embed({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  value: 'sunny day at the beach',
  providerOptions: {
    openai: {
      dimensions: 512, // 임베딩 차원 축소
    },
  },
});
```

### 병렬 요청(Parallel Requests)

`embedMany`는 `maxParallelCalls`로 병렬 처리 개수를 조정해 성능을 최적화할 수 있어.

```ts highlight={"4"}
import { openai } from '@ai-sdk/openai';
import { embedMany } from 'ai';

const { embeddings, usage } = await embedMany({
  maxParallelCalls: 2, // 병렬 요청 제한
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  values: ['sunny day at the beach', 'rainy afternoon in the city', 'snowy night in the mountains'],
});
```

### 재시도(Retries)

`embed`와 `embedMany`는 선택적 `maxRetries: number`를 받아 임베딩 재시도 최대 횟수를 설정할 수 있어.
기본은 2회 재시도(총 3번 시도)고, 0으로 설정해 비활성화할 수 있어.

```ts highlight={"7"}
import { openai } from '@ai-sdk/openai';
import { embed } from 'ai';

const { embedding } = await embed({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  value: 'sunny day at the beach',
  maxRetries: 0, // 재시도 비활성화
});
```

### Abort Signals & Timeouts

`embed`와 `embedMany`는 선택적 `abortSignal: AbortSignal`을 받아 작업 중단이나 타임아웃 설정이 가능해.

```ts highlight={"7"}
import { openai } from '@ai-sdk/openai';
import { embed } from 'ai';

const { embedding } = await embed({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  value: 'sunny day at the beach',
  abortSignal: AbortSignal.timeout(1000), // 1초 후 중단
});
```

### 커스텀 헤더(Custom Headers)

`embed`와 `embedMany`는 선택적 `headers: Record<string, string>`를 받아 요청 헤더를 커스터마이즈할 수 있어.

```ts highlight={"7"}
import { openai } from '@ai-sdk/openai';
import { embed } from 'ai';

const { embedding } = await embed({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  value: 'sunny day at the beach',
  headers: { 'X-Custom-Header': 'custom-value' },
});
```

## 응답 정보(Response Information)

`embed`와 `embedMany`는 원본 제공자 응답에 접근할 수 있는 정보를 함께 반환해.

```ts highlight={"4,9"}
import { openai } from '@ai-sdk/openai';
import { embed } from 'ai';

const { embedding, response } = await embed({
  model: openai.textEmbeddingModel('text-embedding-3-small'),
  value: 'sunny day at the beach',
});

console.log(response); // Raw provider response
```
