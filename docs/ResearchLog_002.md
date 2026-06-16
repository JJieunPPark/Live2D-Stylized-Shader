# Research Log
Date: 2026-06-16

## Topic
Interactive Anime-Style Rendering Pipeline Research

## Background

앙상블 스타즈(앙스타) MV 제작팀 인터뷰를 분석하였다.
cgworld: https://cgworld.jp/article/202602-mellow-3.html
카카리아 스튜디오 어드벤트 캘린더: https://note.com/happyelements/n/n173ee479a981

기존에는 앙스타의 비주얼 퀄리티가 주로 Toon Shader 또는 라이팅 기술에서 나온다고 생각했으나, 인터뷰를 통해 실제로는 카메라 기반 후처리(Post Process) 연출이 큰 비중을 차지한다는 점을 확인하였다.

특히 MV에서는 실제 3D 공간에 안개(Fog)나 플래시(Flash)를 배치하는 것이 아니라, 렌더링된 결과에 Overlay, Screen, Fog 등의 레이어를 덧입혀 일러스트와 유사한 분위기를 구현하고 있었다.

---

## Findings

### 1. Anime Rendering은 Shader 하나로 이루어지지 않는다

현재 앙스타 MV 파이프라인은 다음과 같이 추정된다.

```text
Toon Shader
↓
Depth Buffer 생성
↓
Camera Based Post Process
↓
Fog
Screen
Overlay
Back Flare
↓
Final Image
```

즉, "예쁜 화면"은 단일 셰이더가 아니라 여러 단계의 조합으로 만들어진다.

---

### 2. Depth Buffer의 활용 가능성

앙스타 인터뷰에서는 다음과 같은 연출이 언급되었다. (출처: 카카리아 스튜디오 어드벤트 캘린더)

- 캐릭터 뒤만 빛나게 하기
- 무대 뒤쪽만 밝게 하기

이는 Depth Buffer를 이용한 후처리 효과일 가능성이 높다.

예시:

```glsl
float fog = smoothstep(5.0, 20.0, depth);

color = mix(color, fogColor, fog);
```

실제 안개를 계산하는 대신 Depth 기반으로 안개를 덧입히는 방식이다.

---

### 3. GPU에서 조건문(if)의 문제

GPU는 대량 병렬 연산에 최적화되어 있다.

따라서

```glsl
if(condition)
```

보다

```glsl
step()
smoothstep()
max()
min()
clamp()
```

등을 이용한 마스크 계산 방식이 선호된다.

예시:

```glsl
float keep = smoothstep(0.2, 0.5, dot(N, L));

highlight *= keep;
```

즉, GPU에게 "판단"을 시키기보다 "계산"을 시키는 방향이 효율적이다.

---

### 4. "생성"보다 "삭제"가 쉬울 수 있다

기존 아이디어:

```text
어디에 하이라이트를 넣을까?
```

새로운 아이디어:

```text
일단 넓게 생성
↓
필요 없는 영역 제거
```

예시:

```text
Anisotropic Highlight 생성
↓
Depth
Normal
View Direction
Shadow Mask
등으로 제거
↓
남은 부분만 사용
```

Anime Style 표현에서는

"빛을 어디에 넣을까"

보다

"빛을 어디에 넣지 않을까"

가 더 안정적일 수 있다는 가능성을 발견하였다.

---

### 5. LUT(Texture Lookup Table)의 중요성

그래픽스에서는 복잡한 수식을 매 프레임 계산하지 않고,

미리 계산된 결과를 텍스처 형태로 저장하여 사용한다.

예시:

- Ramp Texture
- BRDF LUT
- Shadow LUT

즉,

```text
수식
↓
미리 계산
↓
텍스처 저장
↓
실시간 Lookup
```

방식이다.

---

## New Idea

### Candidate-Based Anime Rendering

기존 방식:

```text
실시간으로 정확한 BRDF 계산
```

새로운 방식:

```text
하이라이트 후보
반사광 후보
그림자 후보
재질 후보
↓
텍스처로 저장
↓
실시간 조건으로 제거
↓
남은 결과 합성
```

예시:

```text
Hair Highlight Candidate
+
Face Shadow Candidate
+
Rim Light Candidate

↓

Depth
Normal
View Direction
Light Direction

↓

조건에 맞지 않는 후보 제거

↓

최종 출력
```

---

## Candidate Removal System (Draft)

```text
Toon Shader
↓
Highlight Candidate Texture
↓
View Mask
↓
Light Mask
↓
Depth Mask
↓
Shadow Mask
↓
Overlay / Screen Blend
↓
Final Image
```

핵심 아이디어:

> "빛을 계산해서 만드는 것이 아니라,
> 후보를 준비한 뒤 조건에 따라 제거한다."

---

## Open Questions

1. Hair Direction Map을 이용한 Anime Highlight 생성 가능성
2. Anisotropic Reflection을 LUT 기반으로 단순화할 수 있는가
3. Depth 기반 Anime Post Process를 Interactive Environment에서 사용할 수 있는가
4. Candidate Removal 방식이 기존 Toon Shader보다 효율적인가
5. Live2D 스타일의 자동 연출 시스템 구축 가능성

---

## Conclusion

오늘 연구를 통해 NPR(Non-Photorealistic Rendering)의 핵심은 복잡한 물리 기반 계산이 아니라,

- 미리 계산하기
- 텍스처화하기
- 조건으로 제거하기
- 후처리 활용하기

에 가까울 수 있다는 점을 확인하였다.

또한 연구 방향이 단순 Toon Shader 구현에서

**"Interactive Anime Rendering System"**

쪽으로 이동하고 있음을 확인하였다.

### Current Research Direction

```text
Toon Shader
↓
Depth / Normal Buffer
↓
Candidate Texture System
↓
Conditional Removal
↓
Anime Post Process
↓
Interactive Illustration Rendering
```
