# Message Metadata

메시지 메타데이터를 쓰면 메시지 단위로 커스텀 정보를 붙일 수 있어. 타임스탬프, 모델 정보, 토큰 사용량, 사용자 컨텍스트 같은 메시지 레벨 데이터를 추적할 때 유용해.

## 개요

메시지 메타데이터는 [데이터 파트](/docs/ai-sdk-ui/streaming-data)와 달리 메시지 콘텐츠의 일부가 아니라 메시지 레벨에 붙는다는 점이 달라. 데이터 파트가 메시지를 구성하는 동적 콘텐츠에 적합하다면, 메타데이터는 메시지 자체에 대한 정보를 담기에 딱 좋아.

## 시작하기

타임스탬프와 모델 정보를 추적하기 위해 메시지 메타데이터를 사용하는 간단한 예시야.

### 메타데이터 타입 정의

먼저, 타입 안전성을 위해 메타데이터 타입을 정의해:

```tsx filename="app/types.ts"
import { UIMessage } from 'ai';
import { z } from 'zod';

// Define your metadata schema
export const messageMetadataSchema = z.object({
  createdAt: z.number().optional(),
  model: z.string().optional(),
  totalTokens: z.number().optional(),
});

export type MessageMetadata = z.infer<typeof messageMetadataSchema>;

// Create a typed UIMessage
export type MyUIMessage = UIMessage<MessageMetadata>;
```

### 서버에서 메타데이터 전송

스트리밍의 서로 다른 단계에서 메타데이터를 보내려면 `toUIMessageStreamResponse`의 `messageMetadata` 콜백을 사용해:

```ts filename="app/api/chat/route.ts" highlight="11-20"
import { openai } from '@ai-sdk/openai';
import { convertToModelMessages, streamText } from 'ai';
import type { MyUIMessage } from '@/types';

export async function POST(req: Request) {
  const { messages }: { messages: MyUIMessage[] } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse({
    originalMessages: messages, // pass this in for type-safe return objects
    messageMetadata: ({ part }) => {
      // Send metadata when streaming starts
      if (part.type === 'start') {
        return {
          createdAt: Date.now(),
          model: 'gpt-4o',
        };
      }

      // Send additional metadata when streaming completes
      if (part.type === 'finish') {
        return {
          totalTokens: part.totalUsage.totalTokens,
        };
      }
    },
  });
}
```

<Note>
  `messageMetadata`에서 타입 안전한 반환 객체를 활성화하려면,
  네 UIMessage 타입으로 타이핑된 `originalMessages`를 함께 전달해줘.
</Note>

### 클라이언트에서 메타데이터 접근

`message.metadata` 속성을 통해 메타데이터에 접근할 수 있어:

```tsx filename="app/page.tsx" highlight="8,18-23"
'use client';

import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';
import type { MyUIMessage } from '@/types';

export default function Chat() {
  const { messages } = useChat<MyUIMessage>({
    transport: new DefaultChatTransport({
      api: '/api/chat',
    }),
  });

  return (
    <div>
      {messages.map(message => (
        <div key={message.id}>
          <div>
            {message.role === 'user' ? 'User: ' : 'AI: '}
            {message.metadata?.createdAt && (
              <span className="text-sm text-gray-500">
                {new Date(message.metadata.createdAt).toLocaleTimeString()}
              </span>
            )}
          </div>

          {/* Render message content */}
          {message.parts.map((part, index) =>
            part.type === 'text' ? <div key={index}>{part.text}</div> : null,
          )}

          {/* Display additional metadata */}
          {message.metadata?.totalTokens && (
            <div className="text-xs text-gray-400">
              {message.metadata.totalTokens} tokens
            </div>
          )}
        </div>
      ))}
    </div>
  );
}
```

<Note>
  생성 중에 변하는 임의의 데이터를 스트리밍하려면,
  [데이터 파트](/docs/ai-sdk-ui/streaming-data)를 대신 사용하는 걸 고려해줘.
</Note>

## 흔한 사용 사례

메시지 메타데이터는 다음에 적합해:

- 타임스탬프: 메시지가 생성/완료된 시점
- 모델 정보: 사용된 AI 모델
- 토큰 사용량: 비용/사용량 추적
- 사용자 컨텍스트: 사용자 ID, 세션 정보
- 성능 지표: 생성 시간, 첫 토큰까지 시간
- 품질 지표: 종료 사유, 신뢰도 점수

## 함께 보기

- [Chatbot Guide](/docs/ai-sdk-ui/chatbot#message-metadata) - 챗봇 구축 문맥에서의 메시지 메타데이터
- [Streaming Data](/docs/ai-sdk-ui/streaming-data#message-metadata-vs-data-parts) - 데이터 파트와의 비교
- [UIMessage Reference](/docs/reference/ai-sdk-core/ui-message) - UIMessage 타입 레퍼런스 전체
