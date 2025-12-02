# Experiments-Multi-Agent-Continual-Representation-IDS

### 1. 실험 목표

이 프로젝트의 1차 목표는 **온라인·지속 학습(continual learning)** 기반의 지능형 네트워크 침입 탐지 시스템을 만드는 것이다.  
즉,

- 네트워크 트래픽 분포가 시간에 따라 계속 바뀌는 환경(새로운 공격, 공격 강도 변화 등)에서
- 한 번 학습한 모델이 **망각(catastrophic forgetting)** 없이
- 새로운 공격 패턴에 적응하면서도 기존 정상/공격 패턴을 유지할 수 있는지 검증하는 것이 이번 실험의 목적이다.

Toy 단계에서는 **CIC-IDS2017** 데이터셋을 사용해,

1. 클라이언트 측 **비지도 대조 표현 학습 + 적응형 EWC 기반 지속 학습** 모듈,
2. 서버 측 **멀티 에이전트 기반 연합 학습(Federated Learning)** 구조가 실제로 분포 드리프트 상황에서 얼마나 안정적인 성능을 내는지 테스트했다..

---

### 2. 클라이언트: 비지도 대조 표현 학습 + 적응형 EWC

#### 2.1 Unsupervised Contrastive Representation

클라이언트 인코더는 **레이블 없이도 표현을 학습하는 비지도 대조 학습 모듈**이다.

- 입력 샘플 X1 에 **white noise**를 추가해 증강 샘플 X2 를 생성
- (X1, X2)는 “같은 샘플 쌍”으로 간주 → 임베딩 공간에서 가깝게
- (X1, X3)처럼 서로 다른 샘플 쌍은 멀어지도록 학습
- 이렇게 학습된 **contrastive encoder**는 각 네트워크 플로우/패킷 샘플을 **latent vector**로 매핑한다.

이후 이상 탐지는 **단순한 통계 기반 방법**으로 수행한다.

- 여러 샘플을 latent space로 임베딩 한 뒤,
- 각 차원별 평균/표준편차를 추정하고,
- 평균 ± (z-score × 표준편차)를 크게 벗어나는 샘플을 이상치로 간주
- 구현 상으로는 k-NN 기반 근접도 + Z-score 기반 anomaly score 를 함께 사용

즉, 복잡한 분류기보다는  
**“좋은 표현(encoder) + 단순한 통계적 감지기”** 구조를 목표로 한다.

#### 2.2 Continual Representation Learning: Adaptive EWC + Rehearsal Buffer

시간이 흐르면서 트래픽 분포가 바뀌면(새로운 공격 등장, 공격 비율 변화 등),  
기존에 학습된 표현이 쉽게 망가질 수 있다. 이를 막기 위해 **EWC(Elastic Weight Consolidation)** 기반 지속 학습을 도입했다.

전체 손실은 아래와 같이 구성한다.

- L = L_const + L\*_EWC
  - L_const : 현재 배치에 대한 contrastive loss
  - L\*_EWC : 과거 task 에서 중요했던 파라미터를 보존하기 위한 EWC 항

기본 EWC는 **Fisher information의 대각 성분**을 이용해  
“과거 task 에서 중요했던 파라미터”가 크게 변하지 않도록 제약을 건다.

우리는 여기에 **Adaptive EWC** 아이디어를 추가했다.

- 현재 contrastive loss 가 과거 기준보다 커지면  
  → 데이터 분포가 드리프트했다고 가정  
  → EWC 계수(λ)를 키워서 파라미터 변화에 더 강한 제약을 건다.
- 반대로 loss 증가가 크지 않다면  
  → EWC를 상대적으로 완화하여 새로운 패턴을 더 잘 받아들이도록 한다.

또한 **Rehearsal Buffer**를 함께 사용한다.

- 과거 구간에서 일부 샘플을 버퍼에 저장
- 새로운 phase 학습 시 현재 데이터 + 버퍼 데이터를 함께 학습

이를 통해 분포 드리프트 상황에서도  
**이전 정상/공격 패턴을 완전히 잊어버리지 않도록** 돕는다.

---

### 3. 서버: Federated Multi-Agent 구조

서버는 여러 개의 네트워크/도메인을 하나의 연합 학습 프레임워크 안에서 관리한다.

- 구조: **1개 중앙 서버 + 4개 클라이언트**
  - Corporation Network
  - Home Network
  - Drone Network
  - Robotics Network
- 토이 단계에서는 **CIC-IDS2017 데이터를 4부분으로 나누어**  
  위 4개의 네트워크 도메인을 가상으로 구성했다.
- 각 클라이언트는 로컬에서 비지도 대조 학습 + 적응형 EWC 학습을 수행한 후,
  중앙 서버로 모델 가중치를 전송한다.
- 중앙 서버는 현재 단계에서는 **단순 평균(FedAvg)** 방식으로 가중치를 집계한다.

이 실험의 목적은,  
**멀티 에이전트(멀티 클라이언트) 구조 위에서 continual representation learning 이 정상적으로 동작하는지**를 검증하는 것이다.  
다음 단계에서는 실제로 서로 다른 도메인의 데이터(기업/가정/드론/로보틱스/저지연 네트워크 시뮬레이터)를 클라이언트별로 분리해서 사용할 계획이다.

---

### 4. Toy Example: CIC-IDS2017 실험 설정

#### 4.1 Dataset

- **CIC-IDS2017**
  - 약 420,000개 이상의 네트워크 플로우/패킷 기반 샘플
  - 각 샘플은 다수의 통계/프로토콜 피처로 구성된 tabular 데이터

#### 4.2 4-Phase Continual Learning 시나리오

실험에서는 데이터셋을 **4개의 Phase**로 나누어,  
시간에 따라 트래픽 분포가 변하는 상황을 인위적으로 구성했다.

- Phase 1: 정상(benign) 위주의 구간
- Phase 2: 공격(attack) 비율이 높아진 구간
- Phase 3: “heavy attack” 구간 (공격 비율이 매우 높은 극단 상황)
- Phase 4: 다시 정상 위주의 구간으로 복귀

각 Phase마다:

- 각 클라이언트가 **3 local epochs** 동안 로컬 학습을 수행
- 이후 중앙 서버에서 모델을 집계(FedAvg)하는 과정을 반복

#### 4.3 평가 방법

- 각 Phase 종료 시점에 해당 구간의 데이터로 성능 평가
- 주요 지표:
  - Accuracy (정확도)
  - FAR (False Alarm Rate, 오탐률)
- 특히 Phase 3 (heavy attack) 구간에서:
  - 모델이 분포 드리프트에 얼마나 취약한지,
  - continual learning 기법 도입 시 성능 붕괴가 얼마나 완화되는지
  를 비교하는 데 초점을 맞췄다.

---

### 5. 비교한 모델들

동일한 실험 설정에서 아래 모델들을 비교했다.

1. **MLP (Multi-Layer Perceptron)**
   - 가장 단순한 fully-connected 기반 베이스라인

2. **Transformer**
   - 시퀀스/시계열 구조를 반영하는 기본 Transformer 인코더

3. **Transformer + Rehearsal Buffer**
   - 현재 phase 데이터와 과거 buffer 데이터를 함께 학습하는 단순 재현(rehearsal) 방식

4. **Transformer + EWC**
   - 고전적인 EWC 기반 continual learning 적용

5. **Transformer + Adaptive EWC + Rehearsal Buffer (제안 방식)**
   - 손실 변화에 따라 EWC 계수를 조정하는 Adaptive EWC
   - Rehearsal buffer를 함께 사용해 과거 패턴 보존 강화

---

### 6. 주요 결과

#### 6.1 Heavy Attack Phase에서의 성능 비교

- **MLP**
  - 초기 Phase에서는 양호한 정확도
  - **Phase 3(heavy attack)에서 정확도가 약 17% 수준까지 급락**
  - 사실상 정상/공격을 거의 구분하지 못하는 수준으로,  
    분포 드리프트에 매우 취약함을 보여줌

- **Transformer**
  - MLP보다 분포 변화에 강인
  - Phase 3에서도 **약 60%대 중반(≈66%) 정확도**를 유지
  - 그럼에도 불구하고 Phase 1 대비 성능 하락 폭이 큰 편

- **Transformer + Rehearsal Buffer**
  - 기본 Transformer 대비 **성능 하락 폭 감소**
  - 과거 샘플을 함께 학습함으로써 성능 붕괴를 완화하지만,
  - 극단적인 공격 비율 변화 구간에서는 여전히 정확도 흔들림 존재

- **Transformer + EWC**
  - Rehearsal과 유사하게 **성능 안정성 향상**
  - 하지만 고정된 EWC 계수 때문에  
    큰 분포 변화에 충분히 유연하게 적응하지는 못함

- **Transformer + Adaptive EWC + Rehearsal Buffer (제안 방식)**
  - 전 Phase에 걸쳐 **가장 안정적인 Accuracy/FAR**를 보였다.
  - 특히 Phase 3(heavy attack)에서의 정확도 감소 폭을  
    **약 9% 이내**로 억제하는 데 성공했다.
  - 새로운 공격 분포에 적응하면서도,  
    이전 정상/공격 패턴에 대한 표현을 비교적 잘 유지했다.

#### 6.2 요약 인사이트

- 단순 MLP는 심한 분포 드리프트가 발생하는 IDS 환경에는 적합하지 않다.
- Transformer 인코더만으로도 어느 정도의 견고함은 확보되지만,
  극단적인 공격 구간에서는 여전히 catastrophic forgetting 문제가 남아 있다.
- Rehearsal 또는 기본 EWC만으로는 성능 붕괴를 완전히 막지 못한다.
- **Adaptive EWC + Rehearsal Buffer 조합**은
  - 분포 변화(손실 증가)에 따라 정규화 세기를 자동 조절하고,
  - 과거 샘플을 함께 학습에 사용함으로써,
  **온라인·지속 학습 IDS에 필요한 “성능 유지력”을 가장 잘 보여준 방식**으로 확인되었다.

---

### 7. 한계점과 향후 계획

이번 실험을 통해 얻은 한계점과 다음 단계 계획은 아래와 같다.

1. **진짜 멀티-도메인 FL로 확장**
   - 현재는 CIC-IDS2017 하나를 4등분해서 4 클라이언트로 사용
   - 다음 단계에서는 기업/가정/드론/로보틱스/저지연 네트워크 시뮬레이터 등  
     서로 다른 도메인의 데이터를 클라이언트별로 분리하여  
     **이기종 네트워크 환경에서의 연합·지속 학습 성능**을 평가할 예정

2. **Contrastive Module 다양화**
   - 현재는 이미지 도메인에서 쓰이던 BYOL 스타일 모듈을 tabular 데이터에 적용
   - InfoNCE 기반 모듈 등 **tabular 데이터에 더 적합한 대조 학습 구조**를 적용하여  
     표현 품질, 이상 탐지 성능, 학습 안정성을 비교할 계획

3. **고급 Aggregation 방법 탐색**
   - 현재 중앙 서버는 단순 평균(FedAvg)을 사용
   - 각 클라이언트의 데이터 양, 손실, 도메인 특성을 반영하는  
     **가중 합(weighted sum) 기반 집계 방식**으로 확장 예정

4. **평가 전략 고도화 및 시뮬레이터 결합**
   - Train/Test split 기반의 정적 평가뿐 아니라,
   - 시간에 따라 공격 시나리오가 바뀌는 **continual evaluation 시나리오**를 확장
   - 구축 중인 여러 네트워크 시뮬레이터에 다양한 공격 스크립트를 주입해,
     **“훈련 데이터에 없던 공격”에 대한 일반화 성능**까지 평가하는 것을 목표로 한다.
