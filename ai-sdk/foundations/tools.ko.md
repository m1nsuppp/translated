# 도구 (Tools)

LLM(대규모 언어 모델)은 생성 능력이 뛰어나나, 이산적 작업(예: 수학)이나 외부 세계 상호작용(예: 날씨 조회)에는 한계가 있음. 이를 보완하기 위해 도구를 사용함.

도구는 LLM이 호출할 수 있는 "행동"임. 도구 실행 결과는 다음 응답을 생성할 때 고려하도록 LLM에 다시 전달 가능함.

예: “런던 날씨 알려줘” 요청 시 날씨 도구가 있으면, 모델은 런던을 인자로 도구를 호출함. 도구는 실제 날씨 데이터를 가져와 반환하고, 모델은 그 정보를 응답에 반영함.

## 도구란 무엇인가?

도구는 특정 작업을 수행하기 위해 모델이 호출할 수 있는 객체임. `tools` 파라미터로 하나 이상 도구를 전달하면 [`generateText`](/docs/reference/ai-sdk-core/generate-text) 및 [`streamText`](/docs/reference/ai-sdk-core/stream-text)에서 사용 가능함.

도구는 보통 다음 프로퍼티로 구성됨:

- **`description`**: 선택. 도구가 언제 선택될지에 영향을 주는 설명.
- **`inputSchema`**: 도구 실행에 필요한 입력을 정의하는 [Zod 스키마](/docs/foundations/tools#schema-specification-and-validation-with-zod) 또는 [JSON 스키마](/docs/reference/ai-sdk-core/json-schema). 모델이 이 스키마를 참조해 호출 파라미터를 생성하며, 실행 전 검증에도 사용됨.
- **`execute`**: 선택. 도구 호출 인자를 받아 실제로 동작하는 비동기 함수.

<Note>
  `streamUI`는 React 컴포넌트를 반환하는 `generate` 함수를 갖는 UI 생성 도구를 사용함.
</Note>

모델이 도구 사용을 결정하면 도구 호출을 생성함. `execute`가 있는 도구는 자동 실행되며, 그 결과는 도구 결과 객체로 반환됨. 이 결과를 [multi-step calls](/docs/ai-sdk-core/tools-and-tool-calling#multi-step-calls)를 통해 `streamText`/`generateText`가 자동으로 모델에 다시 주입할 수 있음.

## 스키마 (Schemas)

스키마는 도구의 파라미터를 정의하고, [도구 호출](/docs/ai-sdk-core/tools-and-tool-calling)을 검증하는 데 사용됨.

AI SDK는 두 가지 방식을 지원함:
- 원시 JSON 스키마([`jsonSchema`](/docs/reference/ai-sdk-core/json-schema))
- [Zod](https://zod.dev/) 스키마(직접 또는 [`zodSchema`](/docs/reference/ai-sdk-core/zod-schema)로)

[Zod](https://zod.dev/)는 널리 쓰이는 TypeScript 스키마 검증 라이브러리임. 설치 방법:

<Tabs items={['pnpm', 'npm', 'yarn', 'bun']}>
  <Tab>
    <Snippet text="pnpm add zod" dark />
  </Tab>
  <Tab>
    <Snippet text="npm install zod" dark />
  </Tab>
  <Tab>
    <Snippet text="yarn add zod" dark />
  </Tab>

  <Tab>
    <Snippet text="bun add zod" dark />
  </Tab>
</Tabs>

예시 Zod 스키마:

```ts
import z from 'zod';

const recipeSchema = z.object({
  recipe: z.object({
    name: z.string(),
    ingredients: z.array(
      z.object({
        name: z.string(),
        amount: z.string(),
      }),
    ),
    steps: z.array(z.string()),
  }),
});
```

<Note>
  스키마는 구조화 출력 생성에도 사용 가능함: [`generateObject`](/docs/reference/ai-sdk-core/generate-object), [`streamObject`](/docs/reference/ai-sdk-core/stream-object).
</Note>

## 툴킷 (Toolkits)

실무에서는 도메인 특화 도구 + 범용 도구를 혼합해 쓰는 경우가 많음. 다음 제공사들이 **툴킷** 형태로 사전 구축 도구를 제공함:

- **[agentic](https://docs.agentic.so/marketplace/ts-sdks/ai-sdk)** — 20개+ 도구 모음. [Exa](https://exa.ai/), [E2B](https://e2b.dev/) 등 외부 API 연결 중심.
- **[browserbase](https://docs.browserbase.com/integrations/vercel/introduction#vercel-ai-integration)** — 헤드리스 브라우저 실행 도구.
- **[browserless](https://docs.browserless.io/ai-integrations/vercel-ai-sdk)** — 브라우저 자동화(셀프 호스트/클라우드) + AI 연동.
- **[Stripe agent tools](https://docs.stripe.com/agents?framework=vercel)** — Stripe 상호작용 도구.
- **[StackOne ToolSet](https://docs.stackone.com/agents/typescript/frameworks/vercel-ai-sdk)** — 수백 개 [엔터프라이즈 SaaS](https://www.stackone.com/integrations) 에이전틱 통합.
- **[Toolhouse](https://docs.toolhouse.ai/toolhouse/toolhouse-sdk/using-vercel-ai)** — 25개+ 액션을 3줄로 호출.
- **[Agent Tools](https://ai-sdk-agents.vercel.app/?item=introduction)** — 에이전트용 도구 모음.
- **[AI Tool Maker](https://github.com/nihaocami/ai-tool-maker)** — OpenAPI 스펙에서 AI SDK 도구를 생성하는 CLI.
- **[Composio](https://docs.composio.dev/providers/vercel)** — GitHub/Gmail/Salesforce 등 250+ 도구 제공. [목록](https://composio.dev/tools).
- **[Interlify](https://www.interlify.com/docs/integrate-with-vercel-ai)** — API를 도구로 변환해 백엔드 연결을 신속화.
- **[JigsawStack](http://www.jigsawstack.com/docs/integration/vercel)** — 30+ 특화 소형 모델 기반 도구.

<Note>
  AI SDK 호환 오픈소스 도구/라이브러리 보유 시, [PR 등록](https://github.com/vercel/ai/pulls) 환영함.
</Note>

## 더 알아보기 (Learn more)

자세한 내용은 AI SDK 코어 문서의 [Tool Calling](/docs/ai-sdk-core/tools-and-tool-calling) 및 [Agents](/docs/foundations/agents) 참고 바람.
