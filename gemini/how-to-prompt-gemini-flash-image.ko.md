# 이미지 편집을 위한 프롬프트 가이드 (Gemini 2.5 Flash Image)

Gemini 2.5 Flash Image는 텍스트와 이미지(하나 이상)를 함께 입력해 편집, 합성, 스타일 변환을 수행하는 데 강력해.
아래 템플릿과 예시를 참고해서 더 정확하고 일관된 결과를 만들어봐.

## 1) 이미지 편집: 요소 추가/제거/수정

이미지의 스타일·조명·원근을 고려해 자연스럽게 요소를 추가/삭제/수정할 수 있어. 시리즈 이미지에서도 캐릭터 일관성을 유지하도록 도와줘.

- **Template**

```text
Using the provided image of [subject], please [add/remove/modify] [element] to/from the scene. Ensure the change is [description of how the change should integrate].
```

- **Example prompt (영어 유지)**

```text
Using the provided image of my cat, please add a small, knitted wizard hat on its head. Make it look like it's sitting comfortably and matches the soft lighting of the photo.
```

## 2) 인페인팅: 특정 영역만 편집

이미지의 나머지 부분은 그대로 두고, 지정한 영역만 변경하도록 지시할 수 있어.

- **Template**

```text
Using the provided image, change only the [specific element] to [new element/description]. Keep everything else in the image exactly the same, preserving the original style, lighting, and composition.
```

- **Example prompt (영어 유지)**

```text
Using the provided image of a living room, change only the blue sofa to be a vintage, brown leather chesterfield sofa. Keep the rest of the room, including the pillows on the sofa and the lighting, unchanged.
```

## 3) 스타일 변환 (Style transfer)

참고 이미지를 특정 예술가/양식의 스타일로 재구성하도록 요청할 수 있어. 원본 구도는 유지하면서 표현만 바꾸는 방식이야.

- **Template**

```text
Transform the provided photograph of [subject] into the artistic style of [artist/art style]. Preserve the original composition but render it with [description of stylistic elements].
```

- **Example prompt (영어 유지)**

```text
Transform the provided photograph of a modern city street at night into the artistic style of Vincent van Gogh's 'Starry Night'. Preserve the original composition of buildings and cars, but render all elements with swirling, impasto brushstrokes and a dramatic palette of deep blues and bright yellows.
```

## 4) 고급 합성: 여러 이미지 결합

여러 이미지를 맥락으로 제공해 새로운 합성 장면을 생성할 수 있어. 제품 목업이나 크리에이티브 콜라주에 좋아.

- **Template**

```text
Create a new image by combining the elements from the provided images. Take the [element from image 1] and place it with/on the [element from image 2]. The final image should be a [description of the final scene].
```

- **Example prompt (영어 유지)**

```text
Create a professional e-commerce fashion photo. Take the blue floral dress from the first image and let the woman from the second image wear it. Generate a realistic, full-body shot of the woman wearing the dress, with the lighting and shadows adjusted to match an outdoor environment.
```

## 베스트 프랙티스 (Best practices)

- **구체적으로 작성하기**: “판타지 갑옷” 대신 세부 묘사(재질, 패턴, 형태, 색감)를 명시해.
- **캐릭터 일관성 유지**: 반복 편집으로 특징이 흐트러지면, 상세 설명을 포함해 새 대화로 리셋해.
- **맥락과 의도 제공**: “고급 미니멀 스킨케어 브랜드용 로고”처럼 목적을 설명하면 결과가 더 정확해져.
- **점진적 개선**: 한 번에 완벽을 기대하지 말고, “조명을 조금 더 따뜻하게” 같은 후속 지시로 다듬어.
- **의미적 네거티브 프롬프트**: “차량 없음” 대신, “교통 흔적이 없는 텅 빈 거리”처럼 원하는 장면을 긍정 서술로 표현해.
- **종횡비 관리**: 입력 이미지 비율을 유지하려면 프롬프트에 명시해. 여러 이미지를 입력하면 보통 마지막 이미지의 비율을 따르니, 특정 비율이 필요하면 그 비율의 참조 이미지를 같이 제공해.
- **카메라/촬영 언어 활용**: wide-angle shot, macro shot, low-angle, 85mm portrait, Dutch angle 등으로 구도와 렌즈 느낌을 제어해.
