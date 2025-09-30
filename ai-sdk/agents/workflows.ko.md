# 워크플로 패턴(Workflow Patterns)

[overview](/docs/agents/overview)의 빌딩 블록을 다음 패턴들과 결합해 에이전트에 구조와 신뢰성을 더해봐.

- [순차 처리(체인)](#sequential-processing-chains) — 정해진 순서로 단계 실행
- [병렬 처리](#parallel-processing) — 독립 작업을 동시에 실행
- [평가/피드백 루프](#evaluator-optimizer) — 결과를 반복적으로 점검·개선
- [오케스트레이션](#orchestrator-worker) — 여러 컴포넌트를 조율
- [라우팅](#routing) — 컨텍스트에 따라 흐름 분기

## 접근 방식 선택하기

아래를 고려해봐:

- **유연성 vs 통제** — LLM에 얼마나 자유를 줄지, 혹은 얼마나 강하게 행동을 제한해야 하는지
- **오류 허용도** — 너의 유스케이스에서 실수의 결과가 얼마나 치명적인지
- **비용 고려** — 더 복잡한 시스템은 보통 더 많은 LLM 호출과 높은 비용을 의미함
- **유지보수성** — 단순한 아키텍처가 디버깅·수정에 유리함

**요구사항을 만족하는 가장 단순한 접근부터 시작**하고, 필요한 경우에만 다음을 추가해 복잡도를 높여:

1. 작업을 명확한 단계로 분해
2. 특정 능력을 위한 도구 추가
3. 품질 관리를 위한 피드백 루프 도입
4. 복잡한 워크플로를 위한 멀티 에이전트 도입

이제 각 패턴의 예시를 보자.

## 패턴과 예시

이 문서의 패턴은 [Anthropic의 효과적인 에이전트 구축 가이드](https://www.anthropic.com/research/building-effective-agents)를 참고해 정리했어. 각 패턴은 작업 수행의 특정 측면을 다루며, 이들을 조합해 복잡한 문제에 대한 신뢰할 수 있는 해법을 만들 수 있어.

## Sequential Processing (Chains)

가장 단순한 워크플로 패턴은 미리 정의된 순서로 단계를 실행해. 각 단계의 출력은 다음 단계의 입력이 되어 연쇄적인 파이프라인을 형성하지. 이 패턴은 콘텐츠 생성 파이프라인이나 데이터 변환처럼 순서가 분명한 작업에 적합해.

```ts
import { generateText, generateObject } from 'ai';
import { z } from 'zod';

async function generateMarketingCopy(input: string) {
  const model = 'openai/gpt-4o';

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

  // 품질 기준 미달 시, 더 구체적으로 재생성
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

## Routing

이 패턴은 컨텍스트와 중간 결과에 따라 워크플로의 경로를 선택하도록 모델에 맡겨. 모델이 라우터처럼 동작해, 워크플로의 서로 다른 분기를 오가게 하지. 입력이 다양하고 각기 다른 처리 접근이 필요한 경우 적합해. 아래 예시에서는 첫 번째 LLM 호출 결과에 따라 두 번째 호출의 모델 크기와 시스템 프롬프트가 결정돼.

```ts
import { generateObject, generateText } from 'ai';
import { z } from 'zod';

async function handleCustomerQuery(query: string) {
  const model = 'openai/gpt-4o';

  // 1단계: 문의 유형 분류
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

  // 분기 처리: 유형/복잡도에 따라 모델과 시스템 프롬프트 결정
  const { text: response } = await generateText({
    model: classification.complexity === 'simple' ? 'openai/gpt-4o-mini' : 'openai/o4-mini',
    system: {
      general: 'You are an expert customer service agent handling general inquiries.',
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

## Parallel Processing

작업을 독립적인 서브태스크로 쪼개 동시에 실행해. 구조화된 워크플로의 장점을 유지하면서 효율을 높일 수 있어. 예를 들어 여러 문서를 병렬로 분석하거나, 하나의 입력에 대해 보안/성능/유지보수성 등 여러 관점을 동시에 처리할 수 있어.

```ts
import { generateText, generateObject } from 'ai';
import { z } from 'zod';

// 예시: 전문 리뷰어 여러 명이 병렬로 수행하는 코드 리뷰
async function parallelCodeReview(code: string) {
  const model = 'openai/gpt-4o';

  // 병렬 리뷰 실행
  const [securityReview, performanceReview, maintainabilityReview] = await Promise.all([
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

  // 집계: 또 다른 모델을 사용해 요약
  const { text: summary } = await generateText({
    model,
    system: 'You are a technical lead summarizing multiple code reviews.',
    prompt: `Synthesize these code review results into a concise summary with key actions:
    ${JSON.stringify(reviews, null, 2)}`,
  });

  return { reviews, summary };
}
```

## Orchestrator-Worker

오케스트레이터(주 모델)가 여러 전문 워커를 조율하는 패턴이야. 각 워커는 특정 서브태스크에 최적화되고, 오케스트레이터는 전반 컨텍스트를 유지하며 결과의 일관성을 보장해. 다양한 전문성이 필요한 복잡한 작업에 적합해.

```ts
import { generateObject } from 'ai';
import { z } from 'zod';

async function implementFeature(featureRequest: string) {
  // 오케스트레이터: 구현 계획 수립
  const { object: implementationPlan } = await generateObject({
    model: 'openai/o4-mini',
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
    system: 'You are a senior software architect planning feature implementations.',
    prompt: `Analyze this feature request and create an implementation plan:
    ${featureRequest}`,
  });

  // 워커: 계획된 변경사항 실행
  const fileChanges = await Promise.all(
    implementationPlan.files.map(async (file) => {
      // 변경 유형에 특화된 워커 시스템 프롬프트
      const workerSystemPrompt = {
        create:
          'You are an expert at implementing new files following best practices and project patterns.',
        modify:
          'You are an expert at modifying existing code while maintaining consistency and avoiding regressions.',
        delete: 'You are an expert at safely removing code while ensuring no breaking changes.',
      }[file.changeType];

      const { object: change } = await generateObject({
        model: 'openai/gpt-4o',
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

## Evaluator-Optimizer

평가 단계를 명시적으로 넣어 중간 결과의 품질을 점검하고, 기준 미달 시 파라미터를 조정해 재시도하거나 보정 작업을 수행하도록 해. 이렇게 하면 스스로 품질을 높이고 오류를 회복할 수 있는 견고한 워크플로가 돼.

```ts
import { generateText, generateObject } from 'ai';
import { z } from 'zod';

async function translateWithFeedback(text: string, targetLanguage: string) {
  let currentTranslation = '';
  let iterations = 0;
  const MAX_ITERATIONS = 3;

  // 초기 번역
  const { text: translation } = await generateText({
    model: 'openai/gpt-4o-mini', // 첫 시도는 작은 모델 사용
    system: 'You are an expert literary translator.',
    prompt: `Translate this text to ${targetLanguage}, preserving tone and cultural nuances:
    ${text}`,
  });

  currentTranslation = translation;

  // 평가-개선 루프
  while (iterations < MAX_ITERATIONS) {
    // 현재 번역 평가
    const { object: evaluation } = await generateObject({
      model: 'openai/gpt-4o', // 평가는 더 큰 모델 사용
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

    // 기준 충족 여부 확인
    if (
      evaluation.qualityScore >= 8 &&
      evaluation.preservesTone &&
      evaluation.preservesNuance &&
      evaluation.culturallyAccurate
    ) {
      break;
    }

    // 피드백 기반 개선 번역 생성
    const { text: improvedTranslation } = await generateText({
      model: 'openai/gpt-4o', // 더 큰 모델 사용
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
