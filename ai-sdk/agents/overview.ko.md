# 에이전트
에이전트는 **도구**를 **루프**에서 사용하여 작업을 수행하는 **대규모 언어 모델(LLM)**입니다.
이러한 구성 요소들이 함께 작동합니다:
- **LLM**은 입력을 처리하고 다음 동작을 결정합니다
- **도구**는 텍스트 생성을 넘어 기능을 확장합니다(파일 읽기, API 호출, 데이터베이스 쓰기)
- **루프**는 다음을 통해 실행을 조율합니다:
  - **컨텍스트 관리** - 대화 기록을 유지하고 각 단계에서 모델이 보는 것(입력)을 결정
  - **중지 조건** - 루프(작업)가 완료되는 시점을 결정
## Agent 클래스
Agent 클래스는 이 세 가지 구성 요소를 처리합니다. 다음은 루프에서 여러 도구를 사용하여 작업을 수행하는 에이전트입니다:
```ts
import { Experimental_Agent as Agent, stepCountIs, tool } from 'ai';
import { z } from 'zod';
const weatherAgent = new Agent({
  model: 'openai/gpt-4o',
  tools: {
    weather: tool({
      description: 'Get the weather in a location (in Fahrenheit)',
      inputSchema: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }) => ({
        location,
        temperature: 72 + Math.floor(Math.random() * 21) - 10,
      }),
    }),
    convertFahrenheitToCelsius: tool({
      description: 'Convert temperature from Fahrenheit to Celsius',
      inputSchema: z.object({
        temperature: z.number().describe('Temperature in Fahrenheit'),
      }),
      execute: async ({ temperature }) => {
        const celsius = Math.round((temperature - 32) * (5 / 9));
        return { celsius };
      },
    }),
  },
  stopWhen: stepCountIs(20),
});
const result = await weatherAgent.generate({
  prompt: 'What is the weather in San Francisco in celsius?',
});
console.log(result.text); // agent's final answer
console.log(result.steps); // steps taken by the agent
```
에이전트는 자동으로:
1. `weather` 도구를 호출하여 화씨 온도를 가져옵니다
2. `convertFahrenheitToCelsius`를 호출하여 변환합니다
3. 결과와 함께 최종 텍스트 응답을 생성합니다
Agent 클래스는 루프, 컨텍스트 관리 및 중지 조건을 처리합니다.
## Agent 클래스를 사용하는 이유는?
Agent 클래스는 AI SDK로 에이전트를 구축할 때 권장되는 접근 방식입니다. 그 이유는:
- **보일러플레이트 감소** - 루프와 메시지 배열을 관리합니다
- **재사용성 향상** - 한 번 정의하면 애플리케이션 전체에서 사용할 수 있습니다
- **유지보수 단순화** - 에이전트 구성을 업데이트할 단일 위치
대부분의 사용 사례에서는 Agent 클래스로 시작하세요. 복잡한 구조화된 워크플로우에서 각 단계를 명시적으로 제어해야 할 때는 코어 함수(`generateText`, `streamText`)를 사용하세요.
## 구조화된 워크플로우
에이전트는 유연하고 강력하지만 비결정적입니다. 명시적인 제어 흐름으로 신뢰할 수 있고 반복 가능한 결과가 필요할 때는 다음을 결합한 구조화된 워크플로우 패턴과 함께 코어 함수를 사용하세요:
- 명시적 분기를 위한 조건문
- 재사용 가능한 로직을 위한 표준 함수
- 견고성을 위한 에러 처리
- 예측 가능성을 위한 명시적 제어 흐름
