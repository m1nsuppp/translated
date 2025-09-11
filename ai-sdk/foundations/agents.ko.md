# 에이전트 (Agents)

AI 앱을 만들 때는 **맥락을 이해하고 의미 있는 행동을 수행할 수 있는 시스템**이 필요함. 핵심은 유연성과 통제의 균형을 찾는 것임. 아래는 요구사항에 맞게 능력을 매칭할 수 있도록 다양한 접근/패턴을 정리함.

## 빌딩 블록 (Building Blocks)

### 단일 스텝 LLM 생성 (Single-Step LLM Generation)

가장 기본 블록임. LLM 1회 호출로 응답을 얻는 방식. 분류/텍스트 생성 등 단순 작업에 유용함.

### 도구 사용 (Tool Usage)

계산기, API, DB 같은 도구를 통해 LLM 능력을 확장함. 도구는 모델의 행동을 통제된 방식으로 확장해 줌.

복잡 문제를 풀 때, **LLM이 명시적 순서 지정 없이 여러 스텝에 걸쳐 여러 도구를 호출**할 수 있음. 예: DB 조회 → 계산 → 결과 저장. AI SDK는 `stopWhen` 파라미터로 이런 [멀티스텝 도구 사용](#multi-step-tool-usage)을 단순하게 구현 가능하게 함.

### 멀티 에이전트 시스템 (Multi-Agent Systems)

여러 LLM이 협업하며 각자 복잡 작업의 일부에 특화됨. 개별 컴포넌트는 집중도를 유지하면서 전체적으로 정교한 행동을 구현 가능함.

## 패턴 (Patterns)

다음 빌딩 블록은 워크플로 패턴과 결합되어 복잡도를 관리함:

- [순차 처리](#sequential-processing-chains) — 정해진 순서대로 스텝 실행함
- [병렬 처리](#parallel-processing) — 독립 작업을 동시에 실행함
- [평가/피드백 루프](#evaluator-optimizer) — 결과를 반복적으로 점검/개선함
- [오케스트레이션](#orchestrator-worker) — 여러 컴포넌트를 조율함
- [라우팅](#routing) — 컨텍스트 기반으로 경로를 선택함

## 접근 선택 가이드 (Choosing Your Approach)

다음 요소 고려함:

- **유연성 vs 통제** — 모델의 자유도와 행동 제약의 균형
- **오류 허용도** — 실수의 결과/비용이 큰지 여부
- **비용** — 복잡할수록 LLM 호출 증가 → 비용 증가
- **유지보수** — 단순 구조가 디버깅/수정 용이함

**최소 복잡성으로 시작** 권장. 다음이 필요할 때만 복잡도 추가함:

1. 작업을 명확한 단계로 분해함
2. 특정 능력을 위한 도구를 추가함
3. 품질 관리를 위한 피드백 루프를 도입함
4. 복잡한 워크플로에는 멀티 에이전트를 도입함

아래 예제로 패턴을 살펴봄.

## 패턴과 예제 (Patterns with Examples)

아래 패턴은 [Anthropic의 가이드](https://www.anthropic.com/research/building-effective-agents)를 참고해 워크플로를 구성하는 빌딩 블록으로 정리함. 각 패턴은 수행의 특정 단면을 다루며, 적절히 조합하면 복잡 문제에 대한 신뢰할 수 있는 해법을 구성 가능함.

### 순차 처리 (Sequential Processing, Chains)

가장 단순한 워크플로. 스텝을 미리 정의된 순서로 실행함. 각 스텝의 출력이 다음 스텝의 입력이 되어 일련의 연쇄를 이룸. 콘텐츠 생성 파이프라인, 데이터 변환 프로세스 등 명확한 순서를 가진 작업에 적합함.

```ts
import { openai } from '@ai-sdk/openai';
import { generateText, generateObject } from 'ai';
import { z } from 'zod';

async function generateMarketingCopy(input: string) {
  const model = openai('gpt-4o');

  // 1단계: 마케팅 카피 생성
  const { text: copy } = await generateText({
    model,
    prompt: `Write persuasive marketing copy for: ${input}. Focus on benefits and emotional appeal.`,
  });

  // 품질 점검
  const { object: qualityMetrics } = await generateObject({
    model,
    schema: z.object({
      hasCallToAction: z.boolean(),
      emotionalAppeal: z.number().min(1).max(10),
      clarity: z.number().min(1).max(10),
    }),
    prompt: `Evaluate this marketing copy for:
    1. Presence of call to action (true/false)
    2. Emotional appeal (1-10)
    3. Clarity (1-10)

    Copy to evaluate: ${copy}`,
  });

  // 기준 미달 시 재생성
  if (
    !qualityMetrics.hasCallToAction ||
    qualityMetrics.emotionalAppeal < 7 ||
    qualityMetrics.clarity < 7
  ) {
    const { text: improvedCopy } = await generateText({
      model,
      prompt: `Rewrite this marketing copy with:
      ${!qualityMetrics.hasCallToAction ? '- A clear call to action' : ''}
      ${qualityMetrics.emotionalAppeal < 7 ? '- Stronger emotional appeal' : ''}
      ${qualityMetrics.clarity < 7 ? '- Improved clarity and directness' : ''}

      Original copy: ${copy}`,
    });
    return { copy: improvedCopy, qualityMetrics };
  }

  return { copy, qualityMetrics };
}
```

### 라우팅 (Routing)

모델이 컨텍스트/중간 결과를 바탕으로 워크플로 경로를 선택함. 다양한 입력 유형을 다른 경로로 처리하는 데 유용함. 아래 예제는 첫 번째 호출 결과에 따라 두 번째 호출의 모델 크기/시스템 프롬프트를 바꿈.

```ts
import { openai } from '@ai-sdk/openai';
import { generateObject, generateText } from 'ai';
import { z } from 'zod';

async function handleCustomerQuery(query: string) {
  const model = openai('gpt-4o');

  // 1단계: 질의 분류
  const { object: classification } = await generateObject({
    model,
    schema: z.object({
      reasoning: z.string(),
      type: z.enum(['general', 'refund', 'technical']),
      complexity: z.enum(['simple', 'complex']),
    }),
    prompt: `Classify this customer query:
    ${query}

    Determine:
    1. Query type (general, refund, or technical)
    2. Complexity (simple or complex)
    3. Brief reasoning for classification`,
  });

  // 라우팅: 분류 결과에 따라 모델/시스템 프롬프트 설정
  const { text: response } = await generateText({
    model:
      classification.complexity === 'simple'
        ? openai('gpt-4o-mini')
        : openai('o3-mini'),
    system: {
      general:
        'You are an expert customer service agent handling general inquiries.',
      refund:
        'You are a customer service agent specializing in refund requests. Follow company policy and collect necessary information.',
      technical:
        'You are a technical support specialist with deep product knowledge. Focus on clear step-by-step troubleshooting.',
    }[classification.type],
    prompt: query,
  });

  return { response, classification };
}
```

### 병렬 처리 (Parallel Processing)

독립 서브태스크를 동시에 실행해 효율을 높임. 예: 여러 문서 동시 분석, 코드 리뷰의 보안/성능/품질 동시 평가 등.

```ts
import { openai } from '@ai-sdk/openai';
import { generateText, generateObject } from 'ai';
import { z } from 'zod';

// 예: 특화 리뷰어 3종으로 병렬 코드 리뷰
async function parallelCodeReview(code: string) {
  const model = openai('gpt-4o');

  const [securityReview, performanceReview, maintainabilityReview] =
    await Promise.all([
      generateObject({
        model,
        system:
          'You are an expert in code security. Focus on identifying security vulnerabilities, injection risks, and authentication issues.',
        schema: z.object({
          vulnerabilities: z.array(z.string()),
          riskLevel: z.enum(['low', 'medium', 'high']),
          suggestions: z.array(z.string()),
        }),
        prompt: `Review this code:
      ${code}`,
      }),

      generateObject({
        model,
        system:
          'You are an expert in code performance. Focus on identifying performance bottlenecks, memory leaks, and optimization opportunities.',
        schema: z.object({
          issues: z.array(z.string()),
          impact: z.enum(['low', 'medium', 'high']),
          optimizations: z.array(z.string()),
        }),
        prompt: `Review this code:
      ${code}`,
      }),

      generateObject({
        model,
        system:
          'You are an expert in code quality. Focus on code structure, readability, and adherence to best practices.',
        schema: z.object({
          concerns: z.array(z.string()),
          qualityScore: z.number().min(1).max(10),
          recommendations: z.array(z.string()),
        }),
        prompt: `Review this code:
      ${code}`,
      }),
    ]);

  const reviews = [
    { ...securityReview.object, type: 'security' },
    { ...performanceReview.object, type: 'performance' },
    { ...maintainabilityReview.object, type: 'maintainability' },
  ];

  const { text: summary } = await generateText({
    model,
    system: 'You are a technical lead summarizing multiple code reviews.',
    prompt: `Synthesize these code review results into a concise summary with key actions:
    ${JSON.stringify(reviews, null, 2)}`,
  });

  return { reviews, summary };
}
```

### 오케스트레이터-워커 (Orchestrator-Worker)

오케스트레이터(주 모델)가 특화 워커들을 조율함. 각 워커는 특정 서브태스크에 최적화되고, 오케스트레이터는 전체 맥락/일관성을 관리함. 이 패턴은 서로 다른 전문성이 필요한 복잡 작업에 적합함.

```ts
import { openai } from '@ai-sdk/openai';
import { generateObject } from 'ai';
import { z } from 'zod';

async function implementFeature(featureRequest: string) {
  // 오케스트레이터: 구현 계획 수립
  const { object: implementationPlan } = await generateObject({
    model: openai('o3-mini'),
    schema: z.object({
      files: z.array(
        z.object({
          purpose: z.string(),
          filePath: z.string(),
          changeType: z.enum(['create', 'modify', 'delete']),
        }),
      ),
      estimatedComplexity: z.enum(['low', 'medium', 'high']),
    }),
    system:
      'You are a senior software architect planning feature implementations.',
    prompt: `Analyze this feature request and create an implementation plan:
    ${featureRequest}`,
  });

  // 워커: 계획에 따라 변경 수행
  const fileChanges = await Promise.all(
    implementationPlan.files.map(async file => {
      const workerSystemPrompt = {
        create:
          'You are an expert at implementing new files following best practices and project patterns.',
        modify:
          'You are an expert at modifying existing code while maintaining consistency and avoiding regressions.',
        delete:
          'You are an expert at safely removing code while ensuring no breaking changes.',
      }[file.changeType];

      const { object: change } = await generateObject({
        model: openai('gpt-4o'),
        schema: z.object({
          explanation: z.string(),
          code: z.string(),
        }),
        system: workerSystemPrompt,
        prompt: `Implement the changes for ${file.filePath} to support:
        ${file.purpose}

        Consider the overall feature context:
        ${featureRequest}`,
      });

      return {
        file,
        implementation: change,
      };
    }),
  );

  return {
    plan: implementationPlan,
    changes: fileChanges,
  };
}
```

### 평가자-최적화자 (Evaluator-Optimizer)

중간 결과에 대한 전용 평가 스텝을 둬 품질 관리를 수행함. 평가 결과에 따라 계속 진행/파라미터 조정 재시도/교정 조치 중 선택함. 자체 개선/오류 복구가 가능한 견고한 워크플로 구성 가능함.

```ts
import { openai } from '@ai-sdk/openai';
import { generateText, generateObject } from 'ai';
import { z } from 'zod';

async function translateWithFeedback(text: string, targetLanguage: string) {
  let currentTranslation = '';
  let iterations = 0;
  const MAX_ITERATIONS = 3;

  // 초기 번역
  const { text: translation } = await generateText({
    model: openai('gpt-4o-mini'),
    system: 'You are an expert literary translator.',
    prompt: `Translate this text to ${targetLanguage}, preserving tone and cultural nuances:
    ${text}`,
  });

  currentTranslation = translation;

  // 평가-최적화 루프
  while (iterations < MAX_ITERATIONS) {
    const { object: evaluation } = await generateObject({
      model: openai('gpt-4o'),
      schema: z.object({
        qualityScore: z.number().min(1).max(10),
        preservesTone: z.boolean(),
        preservesNuance: z.boolean(),
        culturallyAccurate: z.boolean(),
        specificIssues: z.array(z.string()),
        improvementSuggestions: z.array(z.string()),
      }),
      system: 'You are an expert in evaluating literary translations.',
      prompt: `Evaluate this translation:

      Original: ${text}
      Translation: ${currentTranslation}

      Consider:
      1. Overall quality
      2. Preservation of tone
      3. Preservation of nuance
      4. Cultural accuracy`,
    });

    if (
      evaluation.qualityScore >= 8 &&
      evaluation.preservesTone &&
      evaluation.preservesNuance &&
      evaluation.culturallyAccurate
    ) {
      break;
    }

    const { text: improvedTranslation } = await generateText({
      model: openai('gpt-4o'),
      system: 'You are an expert literary translator.',
      prompt: `Improve this translation based on the following feedback:
      ${evaluation.specificIssues.join('\n')}
      ${evaluation.improvementSuggestions.join('\n')}

      Original: ${text}
      Current Translation: ${currentTranslation}`,
    });

    currentTranslation = improvedTranslation;
    iterations++;
  }

  return {
    finalTranslation: currentTranslation,
    iterationsRequired: iterations,
  };
}
```

## 멀티스텝 도구 사용 (Multi-Step Tool Usage)

워크플로를 사전에 명확히 그리기 어려운 문제에서는, LLM에 저수준 도구 세트를 제공하고 작업을 스스로 잘게 나눠 반복 해결하도록 하는 에이전틱 패턴이 유용함. 이 패턴을 구현하려면 과제가 완료될 때까지 루프에서 LLM을 호출해야 함. AI SDK는 `stopWhen`으로 이를 단순화함.

SDK는 중지 조건 제어권을 제공함. 각 도구 결과 후(각 요청=1스텝) 자동으로 다음 요청을 트리거하며, 모델이 더 이상 도구 호출을 생성하지 않거나 사용자가 정의한 조건(예: `stepCountIs`)을 만족하면 중지함.

<Note>`stopWhen`은 `generateText`와 `streamText` 모두에서 사용 가능함.</Note>

### `stopWhen` 사용 예

수학 문제 풀이 에이전트 예시. [math.js](https://mathjs.org/) 계산기 도구를 호출하여 수식을 평가함.

```ts file='main.ts'
import { openai } from '@ai-sdk/openai';
import { generateText, tool, stepCountIs } from 'ai';
import * as mathjs from 'mathjs';
import { z } from 'zod';

const { text: answer } = await generateText({
  model: openai('gpt-4o-2024-08-06'),
  tools: {
    calculate: tool({
      description:
        'A tool for evaluating mathematical expressions. ' +
        'Example expressions: ' +
        "'1.2 * (2 + 4.5)', '12.7 cm to inch', 'sin(45 deg) ^ 2'.",
      inputSchema: z.object({ expression: z.string() }),
      execute: async ({ expression }) => mathjs.evaluate(expression),
    }),
  },
  stopWhen: stepCountIs(10),
  system:
    'You are solving math problems. ' +
    'Reason step by step. ' +
    'Use the calculator when necessary. ' +
    'When you give the final answer, ' +
    'provide an explanation for how you arrived at it.',
  prompt:
    'A taxi driver earns $9461 per 1-hour of work. ' +
    'If he works 12 hours a day and in 1 hour ' +
    'he uses 12 liters of petrol with a price  of $134 for 1 liter. ' +
    'How much money does he earn in one day?',
});

console.log(`ANSWER: ${answer}`);
```

### 구조화 답변 (Structured Answers)

수학 분석/리포트 생성 같은 작업에서는 최종 출력을 일관 포맷으로 강제하는 게 유용함. **answer 도구**와 `toolChoice: 'required'`를 사용하면 LLM이 answer 도구의 스키마에 맞춘 구조화 출력을 강제하게 됨. answer 도구에는 `execute` 함수가 없으므로, 이를 호출하면 에이전트가 종료됨.

```ts highlight="6,16-29,31,45"
import { openai } from '@ai-sdk/openai';
import { generateText, tool, stepCountIs } from 'ai';
import 'dotenv/config';
import { z } from 'zod';

const { toolCalls } = await generateText({
  model: openai('gpt-4o-2024-08-06'),
  tools: {
    calculate: tool({
      description:
        'A tool for evaluating mathematical expressions. Example expressions: ' +
        "'1.2 * (2 + 4.5)', '12.7 cm to inch', 'sin(45 deg) ^ 2'.",
      inputSchema: z.object({ expression: z.string() }),
      execute: async ({ expression }) => mathjs.evaluate(expression),
    }),
    // answer tool: the LLM will provide a structured answer
    answer: tool({
      description: 'A tool for providing the final answer.',
      inputSchema: z.object({
        steps: z.array(
          z.object({
            calculation: z.string(),
            reasoning: z.string(),
          }),
        ),
        answer: z.string(),
      }),
      // no execute function - invoking it will terminate the agent
    }),
  },
  toolChoice: 'required',
  stopWhen: stepCountIs(10),
  system:
    'You are solving math problems. ' +
    'Reason step by step. ' +
    'Use the calculator when necessary. ' +
    'The calculator can only do simple additions, subtractions, multiplications, and divisions. ' +
    'When you give the final answer, provide an explanation for how you got it.',
  prompt:
    'A taxi driver earns $9461 per 1-hour work. ' +
    'If he works 12 hours a day and in 1 hour he uses 14-liters petrol with price $134 for 1-liter. ' +
    'How much money does he earn in one day?',
});

console.log(`FINAL TOOL CALLS: ${JSON.stringify(toolCalls, null, 2)}`);
```

<Note>
  `generateText`의 구조화 출력을 위해 [`experimental_output`](/docs/ai-sdk-core/generating-structured-data#structured-output-with-generatetext) 설정 사용 가능함.
</Note>

### 전체 스텝 접근 (Accessing all steps)

`stopWhen` 사용 시, 여러 번의 LLM 호출(스텝)로 이어질 수 있음. 응답의 `steps` 프로퍼티로 모든 스텝 정보를 확인 가능함.

```ts highlight="3,9-10"
import { generateText, stepCountIs } from 'ai';

const { steps } = await generateText({
  model: openai('gpt-4o'),
  stopWhen: stepCountIs(10),
  // ...
});

// 모든 도구 호출 추출 예시
const allToolCalls = steps.flatMap(step => step.toolCalls);
```

### 스텝 완료 콜백 (Getting notified on each completed step)

각 스텝 완료 시점을 통지받으려면 `onStepFinish` 콜백 사용함. 한 스텝의 모든 텍스트 델타/도구 호출/도구 결과가 모였을 때 트리거됨.

```tsx highlight="6-8"
import { generateText, stepCountIs } from 'ai';

const result = await generateText({
  model: 'openai/gpt-4.1',
  stopWhen: stepCountIs(10),
  onStepFinish({ text, toolCalls, toolResults, finishReason, usage }) {
    // 예: 히스토리 저장/사용량 기록 등 커스텀 로직
  },
  // ...
});
```
