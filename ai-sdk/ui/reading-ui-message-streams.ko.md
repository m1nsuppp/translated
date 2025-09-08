# Reading UI Message Streams

`UIMessage` 스트림은 전통적인 채팅 사용 사례 밖에서도 유용해. 터미널 UI, 클라이언트에서의 커스텀 스트림 처리, 또는 React Server Components(RSC) 같은 곳에서 소비할 수 있어.

`readUIMessageStream` 헬퍼는 `UIMessageChunk` 객체 스트림을 `UIMessage` 객체의 `AsyncIterableStream`으로 변환해, 메시지가 생성되는 동안 바로바로 처리할 수 있게 해줘.

## 기본 사용법

```tsx
import { openai } from '@ai-sdk/openai';
import { readUIMessageStream, streamText } from 'ai';

async function main() {
  const result = streamText({
    model: openai('gpt-4o'),
    prompt: 'Write a short story about a robot.',
  });

  for await (const uiMessage of readUIMessageStream({
    stream: result.toUIMessageStream(),
  })) {
    console.log('Current message state:', uiMessage);
  }
}
```

## 툴 콜 통합

툴 콜이 포함된 스트리밍 응답을 처리해:

```tsx
import { openai } from '@ai-sdk/openai';
import { readUIMessageStream, streamText, tool } from 'ai';
import { z } from 'zod';

async function handleToolCalls() {
  const result = streamText({
    model: openai('gpt-4o'),
    tools: {
      weather: tool({
        description: 'Get the weather in a location',
        inputSchema: z.object({
          location: z.string().describe('The location to get the weather for'),
        }),
        execute: ({ location }) => ({
          location,
          temperature: 72 + Math.floor(Math.random() * 21) - 10,
        }),
      }),
    },
    prompt: 'What is the weather in Tokyo?',
  });

  for await (const uiMessage of readUIMessageStream({
    stream: result.toUIMessageStream(),
  })) {
    // Handle different part types
    uiMessage.parts.forEach(part => {
      switch (part.type) {
        case 'text':
          console.log('Text:', part.text);
          break;
        case 'tool-call':
          console.log('Tool called:', part.toolName, 'with args:', part.args);
          break;
        case 'tool-result':
          console.log('Tool result:', part.result);
          break;
      }
    });
  }
}
```

## 대화 재개(Resuming Conversations)

이전 메시지 상태에서 스트리밍을 이어서 진행해:

```tsx
import { readUIMessageStream, streamText } from 'ai';

async function resumeConversation(lastMessage: UIMessage) {
  const result = streamText({
    model: openai('gpt-4o'),
    messages: [
      { role: 'user', content: 'Continue our previous conversation.' },
    ],
  });

  // Resume from the last message
  for await (const uiMessage of readUIMessageStream({
    stream: result.toUIMessageStream(),
    message: lastMessage, // Resume from this message
  })) {
    console.log('Resumed message:', uiMessage);
  }
}
```
