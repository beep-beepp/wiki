---
title: "Attention Is All You Need"
parent: Paper Review
grand_parent: Machine Learning
permalink: /ml/paper-review/attention-is-all-you-need
---

정리

### 무엇이 핵심이었나?
기존 NLP 모델은 주로 RNN / LSTM / GRPU 같은 순차 모델을 사용
문제점:
- 문장을 순서대로 처리 -> 느림
- 긴 문장에서 장기 의존성 학습이 어려움
=> 이 논문의 핵심: 순차 모델이 필요 없다. Attention 만으로 충분하다.

### Attention 이란?
한 단어를 이해할 때 문장 내 다른 단어들을 얼마나 참고해야 하는지를 계산하는 매커니즘
예: "나는 바보다" 에서 "나"가 무엇을 가리키는지 판단할 때 "바보"에 높은 attention을 줌

### Transformer 구조
[ Encoder - Decoder 구조 ]
- Encoder: 입력 문장 이해
- Decoder: 출력 문장 생성

[ Encoder ]
1. (input) Embedding - Positional Encoding
    - 단어 위치 정보를 수학적으로 추가
2. Self-Attention
    - 문장 안에서 단어 <> 단어 관계를 계산 ( Q, K, V )
    - 병럴 처리 가능 -> 매우 빠름
3. Multi-Head Attention
    - Attention을 여러 관점(head)에서 동시에 수행
    - 문법, 의미, 거리 등 다양한 관계를 학습
4. Feed Forward Network
    - Attention 결과를 비선형 변환

[ Decoder ]
1. (input) Embedding - Positional Encoding
2. Masked Multi-Head Attention
    - self-attention 을 하되, 미래 단어를 보면 안됨 ( 예측모델을 학습하는 거니까 )
    - 미래 단어를 -무한대로 마스크를 씌움
3. Multi-Head Attention
    - 인코더 정보를 어떻게 활용할지를 학습하는 레이어
    - 인코더 정보를 활용해서 다음에 나올 토큰 확률을 만들 때 이 토큰이 이전 토큰 중 누구를 참고해야 하는지 등 관계 기반 정보를 학습
      ( 비유: 무엇을 볼지 정함, 자료조사 )
4. Feed Forward Network
    - attention 으로 모은 정보를 비선형 변환(ReLU)로 한번 깨고 차원을 늘렸다가 줄이면서 의미 있는 특징을 뽑아낸다
      ( 비유: 무엇을 표현할지 정함, 해석 + 정리 )
5. Linear - Softmax
    - 최종 토큰 반환

> 비고 <
- Add & Norm 구조는?
    : attention, FFN 같은 복잡한 변환을 여러층 이상 반복한 상태로 계속 layer 만 쌓으면
      입력 정보 손실, gradient 손실/폭주가 발생하는데, 이를 방지
    : normalization 을 통해 분포를 정리
    : (비유)
        - sublayer: 이 정보를 이렇게 바꿔보자
        - add: 근데 원래 정보도 같이 들고 가자
        - norm: 다음 단계가 쓰기 좋게 정리하자