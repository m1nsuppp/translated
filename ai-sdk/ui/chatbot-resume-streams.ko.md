# 챗봇 스트림 재개 (Chatbot Resume Streams)

`useChat`은 페이지 새로고침 이후에도 진행 중인 스트림을 재개(resume)할 수 있어. 이 기능을 이용하면 시간이 오래 걸리는 생성 작업을 하는 애플리케이션을 안정적으로 만들 수 있어.

<Note type="warning">
  스트림 재개는 abort 기능과 호환되지 않아. 탭을 닫거나 페이지를 새로고침하면 abort 시그널이 발생해서 재개 메커니즘이 깨져. 애플리케이션에 abort 기능이 필요하다면 `resume: true`를 사용하지 마. 자세한 내용은
  [troubleshooting](/docs/troubleshooting/abort-breaks-resumable-streams)를 참고해.
</Note>

## 스트림 재개가 동작하는 방식

스트림 재개를 사용하려면 애플리케이션에서 메시지와 활성 스트림을 영속화(persist)해야 해. AI SDK는 스토리지에 연결하기 위한 도구를 제공하지만, 스토리지는 직접 구성해야 해.

**AI SDK가 제공하는 것:**

- `useChat`의 `resume` 옵션: 활성 스트림에 자동 재연결
- `consumeSseStream` 콜백: 외부로 나가는 스트림 접근 제공
- 재개(resume) 엔드포인트로의 자동 HTTP 요청

**네가 구축해야 할 것:**

- 각 챗과 스트림의 매핑을 추적하는 스토리지
- UIMessage 스트림을 저장할 Redis
- 두 개의 API 엔드포인트: 스트림 생성용 POST, 재개용 GET
- Redis 스토리지 관리를 위한 [`resumable-stream`](https://www.npmjs.com/package/resumable-stream) 통합

## 사전 준비물

재개 가능한 스트림을 구현하려면 다음이 필요해:

1. **`resumable-stream` 패키지** — 스트림용 publish/subscribe 메커니즘 처리
2. **Redis 인스턴스** — 스트림 데이터 저장(예: [Vercel의 Redis](https://vercel.com/marketplace/redis))
3. **영속성 레이어** — 각 챗에 대해 활성화된 스트림 ID 추적(예: 데이터베이스)

## 구현

### 1. 클라이언트: 스트림 재개 활성화

`useChat` 훅의 `resume` 옵션을 사용해서 스트림 재개를 활성화해. `resume`이 true면, 훅은 마운트 시 해당 챗의 활성 스트림에 자동으로 재연결을 시도해:

```tsx filename="app/chat/[chatId]/chat.tsx"
"use client";

import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport, type UIMessage } from "ai";

export function Chat({
  chatData,
  resume = false,
}: {
  chatData: { id: string; messages: UIMessage[] };
  resume?: boolean;
}) {
  const { messages, sendMessage, status } = useChat({
    id: chatData.id,
    messages: chatData.messages,
    resume, // 스트림 자동 재개 활성화
    transport: new DefaultChatTransport({
      // 각 요청에 챗 ID를 포함해야 해
      prepareSendMessagesRequest: ({ id, messages }) => {
        return {
          body: {
            id,
            message: messages[messages.length - 1],
          },
        };
      },
    }),
  });

  return <div>{/* 여기서 UI를 구성 */}</div>;
}
```

<Note>
  각 요청마다 챗 ID를 보내야 해(`prepareSendMessagesRequest` 참고).
</Note>

`resume`을 활성화하면, `useChat` 훅은 마운트 시 `/api/chat/[id]/stream`에 `GET` 요청을 보내 활성 스트림이 있는지 확인하고 재개해.

이제 먼저, 재개 가능한 스트림을 생성하는 POST 핸들러부터 만들자.

### 2. POST 핸들러 만들기

POST 핸들러는 `consumeSseStream` 콜백을 사용해 재개 가능한 스트림을 생성해:

```ts filename="app/api/chat/route.ts"
import { openai } from "@ai-sdk/openai";
import { readChat, saveChat } from "@util/chat-store";
import {
  convertToModelMessages,
  generateId,
  streamText,
  type UIMessage,
} from "ai";
import { after } from "next/server";
import { createResumableStreamContext } from "resumable-stream";

export async function POST(req: Request) {
  const {
    message,
    id,
  }: {
    message: UIMessage | undefined;
    id: string;
  } = await req.json();

  const chat = await readChat(id);
  let messages = chat.messages;

  messages = [...messages, message!];

  // 이전 활성 스트림을 초기화하고 사용자 메시지를 저장
  saveChat({ id, messages, activeStreamId: null });

  const result = streamText({
    model: openai("gpt-4o-mini"),
    messages: convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse({
    originalMessages: messages,
    generateMessageId: generateId,
    onFinish: ({ messages }) => {
      // 완료 시 활성 스트림 정리
      saveChat({ id, messages, activeStreamId: null });
    },
    async consumeSseStream({ stream }) {
      const streamId = generateId();

      // SSE 스트림으로부터 재개 가능한 스트림 생성
      const streamContext = createResumableStreamContext({ waitUntil: after });
      await streamContext.createNewResumableStream(streamId, () => stream);

      // 활성 스트림 ID를 챗에 저장
      saveChat({ id, activeStreamId: streamId });
    },
  });
}
```

### 3. GET 핸들러 구현

`/api/chat/[id]/stream` 경로에 GET 핸들러를 만들어서 다음을 수행해:

1. 라우트 파라미터에서 챗 ID 읽기
2. 챗 데이터를 로드하여 활성 스트림이 있는지 확인
3. 활성 스트림이 없으면 204(No Content) 반환
4. 기존 스트림이 있으면 재개해서 반환

```ts filename="app/api/chat/[id]/stream/route.ts"
import { readChat } from "@util/chat-store";
import { UI_MESSAGE_STREAM_HEADERS } from "ai";
import { after } from "next/server";
import { createResumableStreamContext } from "resumable-stream";

export async function GET(
  _: Request,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;

  const chat = await readChat(id);

  if (chat.activeStreamId == null) {
    // 활성 스트림이 없으면 No Content 응답
    return new Response(null, { status: 204 });
  }

  const streamContext = createResumableStreamContext({
    waitUntil: after,
  });

  return new Response(
    await streamContext.resumeExistingStream(chat.activeStreamId),
    { headers: UI_MESSAGE_STREAM_HEADERS }
  );
}
```

<Note>
  Next.js의 `after` 함수는 응답을 보낸 뒤에도 작업을 계속 수행할 수 있게 해 줘. 덕분에 재개 가능한 스트림이 초기 응답 후에도 Redis에 유지되어, 나중에 재연결할 수 있어.
</Note>

## 동작 방식

### 요청 라이프사이클

![재개 가능한 스트림 요청의 아키텍처와 라이프사이클 다이어그램](https://e742qlubrjnjqpp0.public.blob.vercel-storage.com/resume-stream-diagram.png)

위 다이어그램은 재개 가능한 스트림의 전체 라이프사이클을 보여줘:

1. **스트림 생성**: 새 메시지를 보낼 때, POST 핸들러는 `streamText`로 응답 생성을 시작해. `consumeSseStream` 콜백은 고유 ID로 재개 가능한 스트림을 만들고, `resumable-stream` 패키지를 통해 Redis에 저장해.
2. **스트림 추적**: 영속성 레이어가 챗 데이터에 `activeStreamId`를 저장해.
3. **클라이언트 재연결**: 클라이언트가 다시 접속(페이지 새로고침)하면, `resume` 옵션이 `/api/chat/[id]/stream`으로 GET 요청을 보내.
4. **스트림 복구**: GET 핸들러가 `activeStreamId`를 확인하고 `resumeExistingStream`으로 재연결해. 활성 스트림이 없으면 204(No Content)를 반환해.
5. **완료 정리**: 스트림이 끝나면 `onFinish` 콜백에서 `activeStreamId`를 `null`로 설정해서 정리해.

## 재개 엔드포인트 커스터마이즈

기본적으로, `useChat` 훅은 재개 시 `/api/chat/[id]/stream`으로 GET 요청을 보냄. `DefaultChatTransport`의 `prepareReconnectToStreamRequest` 옵션을 사용하면 엔드포인트, 인증, 헤더 등을 커스터마이즈할 수 있어:

```tsx filename="app/chat/[chatId]/chat.tsx"
import { useChat } from "@ai-sdk/react";
import { DefaultChatTransport } from "ai";

export function Chat({ chatData, resume }) {
  const { messages, sendMessage } = useChat({
    id: chatData.id,
    messages: chatData.messages,
    resume,
    transport: new DefaultChatTransport({
      // 재연결 설정 커스터마이즈 (선택)
      prepareReconnectToStreamRequest: ({ id }) => {
        return {
          api: `/api/chat/${id}/stream`, // 기본 패턴
          // 다른 패턴 예시:
          // api: `/api/streams/${id}/resume`,
          // api: `/api/resume-chat?id=${id}`,
          credentials: "include", // 쿠키/인증 포함
          headers: {
            Authorization: "Bearer token",
            "X-Custom-Header": "value",
          },
        };
      },
    }),
  });

  return <div>{/* 여기서 UI를 구성 */}</div>;
}
```

이를 통해 다음을 할 수 있어:

- 기존 API 라우트 구조에 맞추기
- 쿼리 파라미터나 커스텀 경로 추가
- 다양한 백엔드 아키텍처와 통합

## 중요 고려사항

- **abort와의 비호환성**: 스트림 재개는 abort 기능과 호환되지 않아. 탭을 닫거나 새로고침하면 abort 시그널이 발생해서 재개 메커니즘이 깨져. 애플리케이션에 abort가 필요하면 `resume: true`를 쓰지 마.
- **스트림 만료**: Redis의 스트림은 일정 시간이 지나면 만료돼(설정은 `resumable-stream` 패키지에서 구성 가능).
- **다중 클라이언트**: 여러 클라이언트가 동일한 스트림에 동시에 연결할 수 있어.
- **에러 처리**: 활성 스트림이 없을 경우 GET 핸들러는 204(No Content)를 반환해.
- **보안**: 스트림 생성과 재개 모두에 대해 적절한 인증/인가를 적용해.
- **레이스 컨디션**: 오래된 스트림을 재개하지 않도록, 새 스트림을 시작할 때 `activeStreamId`를 먼저 초기화해.

<br />
<GithubLink link="https://github.com/vercel/ai/blob/main/examples/next" />
