# Deep Learning Tuning Playbook

*This is not an officially supported Google product.*

**Varun Godbole<sup>&dagger;</sup>, George E. Dahl<sup>&dagger;</sup>, Justin Gilmer<sup>&dagger;</sup>, Christopher J. Shallue<sup>&Dagger;</sup>, Zachary Nado<sup>&dagger;</sup>**


&dagger; Google Research, Brain Team

&Dagger; Harvard University

## Table of Contents

-   [Who is this document for?](#who-is-this-document-for)
-   [Why a tuning playbook?](#why-a-tuning-playbook)
-   [Guide for starting a new project](#guide-for-starting-a-new-project)
    -   [Choosing the model architecture](#choosing-a-model-architecture)
    -   [Choosing the optimizer](#choosing-the-optimizer)
    -   [Choosing the batch size](#choosing-the-batch-size)
    -   [Choosing the initial configuration](#choosing-the-initial-configuration)
-   [A scientific approach to improving model performance](#a-scientific-approach-to-improving-model-performance)
    -   [The incremental tuning strategy](#the-incremental-tuning-strategy)
    -   [Exploration vs exploitation](#exploration-vs-exploitation)
    -   [Choosing the goal for the next round of experiments](#choosing-the-goal-for-the-next-round-of-experiments)
    -   [Designing the next round of experiments](#Designing-the-next-round-of-experiments)
    -   [Determining whether to adopt a training pipeline change or
        hyperparameter
        configuration](#Determining-whether-to-adopt-a-training-pipeline-change-or-hyperparameter-configuration)
    -   [After exploration concludes](#After-exploration-concludes)
-   [Determining the number of steps for each training run](#Determining-the-number-of-steps-for-each-training-run)
    -   [Deciding how long to train when training is not compute-bound](#Deciding-how-long-to-train-when-training-is-not-compute-bound)
    -   [Deciding how long to train when training is compute-bound](#Deciding-how-long-to-train-when-training-is-compute-bound)
-   [Additional guidance for the training pipeline](#Additional-guidance-for-the-training-pipeline)
    -   [Optimizing the input pipeline](#Optimizing-the-input-pipeline)
    -   [Evaluating model performance](Evaluating-model-performance)
    -   [Saving checkpoints and retrospectively selecting the best checkpoint](#Saving-checkpoints-and-retrospectively-selecting-the-best-checkpoint)
    -   [Setting up experiment tracking](#Setting-up-experiment-tracking)
    -   [Batch normalization implementation details](#Batch-normalization-implementation-details)
    -   [Considerations for multi-host pipelines](#Considerations-for-multi-host-pipelines)
-   [FAQs](#faqs)
-   [Acknowledgments](#acknowledgments)
-   [Citing](#citing)
-   [Contributing](#contributing)

## Who is this document for?

이 문서는 딥 러닝 모델의 성능 극대화에 관심 있는 엔지니어와 연구자들을 위한 것입니다. 머신 러닝과 딥 러닝에 대한 기본 지식을 갖춘 개인과 팀 모두에게 유용할 것입니다.

## Why a tuning playbook?

주요 내용:

- **하이퍼파라미터 튜닝** 프로세스에 중점
- 파이프라인 구현 및 **최적화** 등 일부 다룸
- 주로 **지도 학습** 및 유사 문제(예: 자기 지도 학습) **대상**

튜닝 플레이북의 필요성

1) 실제 딥 뉴럴 네트워크 **최적화의 어려움**

  - 많은 노력과 시행착오 필요
  - 효과적인 방법들이 잘 문서화되지 않음


2) 기존 자료의 한계

  - **논문: 최종 결과에 집중, 과정 생략**
  - 실무자: 경험 일반화할 시간 부족
  - 교과서: 실용적 조언보다 이론 중심


3) **포괄적 가이드의 부재**

  - 딥 러닝으로 좋은 결과를 얻는 방법에 대한 종합적 설명 부족
  - **파편화된 정보**: 블로그, 소셜 미디어, 논문 부록 등


저자 소개 및 문서의 목적

- 5명의 경험 많은 딥 러닝 연구원/엔지니어 팀
- 다양한 분야에 딥 러닝 적용 경험
- 목표: 실용적이고 포괄적인 딥 러닝 가이드 제공

문서의 특징

- 저자들의 현재 견해 반영 (객관적 진실 아님)
- 하이퍼파라미터 튜닝에 중점
- 지속적으로 업데이트되는 '살아있는 문서'
- 커뮤니티의 피드백과 기여 환영

## Guide for starting a new project

프로젝트 초기에 내리는 많은 결정들은 상황 변화가 있을 때만 간헐적으로 재검토하면 됩니다. 아래의 가이드는 다음을 전제로 합니다:

- 문제 정의, 데이터 정제 등 필수 작업이 충분히 완료되어 모델 아키텍처와 훈련 구성에 시간을 투자하는 것이 타당한 상태
- 훈련 및 평가를 위한 파이프라인이 이미 구축되어 있으며, 다양한 모델에 대한 훈련 및 예측 작업을 쉽게 실행할 수 있는 상태
- 적절한 평가 지표가 선택되고 구현된 상태. 이 지표들은 실제 배포 환경에서 측정될 것과 최대한 유사해야 함


### Choosing the model architecture

***Summary:*** *새 프로젝트를 시작할 때는 이미 작동하는 모델을 재사용하려 노력하세요.*

-  먼저 잘 확립되고 널리 사용되는 모델 아키텍처를 선택하여 작동시키세요. 나중에 언제든 커스텀 모델을 만들 수 있습니다.
-  모델 아키텍처에는 보통 모델의 크기와 세부 사항을 결정하는 다양한 하이퍼파라미터가 있습니다(예: 레이어 수, 레이어 폭, 활성화 함수 유형).

    - 따라서 아키텍처를 선택한다는 것은 실제로 다양한 모델 군(각 하이퍼파라미터 설정에 따른)을 선택하는 것을 의미합니다.
    - 모델 하이퍼파라미터 선택 문제는 [초기 구성 선택](#choosing-the-initial-configuration)과 [모델 성능 개선을 위한 과학적 접근](#a-scientific-approach-to-improving-model-performance) 섹션에서 다룰 예정입니다.


-  가능하다면 현재 다루고자 하는 문제와 최대한 유사한 문제를 다룬 논문을 찾아 그 모델을 출발점으로 재현해 보세요.


### Choosing the optimizer

***Summary:*** *문제 유형에 가장 적합한(잘 알려진) 최적화 알고리즘부터 시작하세요.*

-   어떤 최적화 알고리즘도 모든 유형의 머신러닝 문제와 모델 아키텍처에 대해 "최고"일 수는 없습니다. 심지어 [최적화 알고리즘의 성능을 비교하는 것조차 어려운 일입니다.](https://arxiv.org/abs/1910.05446).
    🤖
-   새로운 프로젝트를 시작할 때는 잘 알려진 인기 있는 최적화 알고리즘을 사용하는 것이 좋습니다.
    -   가능하다면 같은 유형의 문제에서 가장 많이 사용되는 최적화 알고리즘을 선택하세요.
-   선택한 최적화 알고리즘의 *모든* 하이퍼파라미터에 신경 쓸 준비를 하세요.
    -   하이퍼파라미터가 많은 최적화 알고리즘일수록 최적의 설정을 찾기 위해 더 많은 튜닝 노력이 필요할 수 있습니다.
    -   이는 특히 프로젝트 초기 단계에서 다양한 하이퍼파라미터(예: 아키텍처 하이퍼파라미터)의 최적 값을 찾으려고 할 때 중요합니다. 이때 최적화 알고리즘의 하이퍼파라미터는 불편한(nuisance) 파라미터로 간주됩니다.
        [nuisance parameters](#identifying-scientific-nuisance-and-fixed-hyperparameters).
    -   초기 단계에서는 단순한 최적화 알고리즘(e.g. 고정된 모멘텀을 가진 SGD 또는 고정된 $\epsilon$, $\beta_{1}$, $\beta_{2}$를 가진 Adam)으로 시작하고, 나중에 더 일반적인 최적화 알고리즘으로 전환하는 것이 바람직할 수 있습니다.
-   우리가 선호하는 잘 알려진 최적화 알고리즘으로는 다음이 포함됩니다(하지만 여기에만 국한되지 않음):
    -   [SGD with momentum](#what-are-the-update-rules-for-all-the-popular-optimization-algorithms)
        (Nesterov 변형을 선호함)
    -   [Adam and NAdam](#what-are-the-update-rules-for-all-the-popular-optimization-algorithms),
        이들은 모멘텀이 있는 SGD보다 더 일반적인 알고리즘입니다. 참고로, Adam에는 조정 가능한 4개의 하이퍼파라미터가 있으며, [이들 모두가 중요할 수 있습니다!](https://arxiv.org/abs/1910.05446)!
        -   See
            [Adam's hyperparameters 튜닝 방법 자료](#how-should-adams-hyperparameters-be-tuned)

### Choosing the batch size

***Summary:*** *배치 크기는 훈련 속도를 좌우하며, 검증 세트 성능을 직접적으로 조정하기 위한 수단으로 사용해서는 안 됩니다. 이상적인 배치 크기는 사용 가능한 하드웨어가 지원하는 최대 크기일 때가 많습니다.*

-  배치 크기는 **훈련 시간**과 **컴퓨팅 자원 소비**를 결정하는 중요한 요소입니다.
배치 크기를 늘리면 훈련 시간이 줄어들 수 있습니다. 이는 다음과 같은 이유로 매우 유익할 수 있습니다:
    -   제한된 시간 내에 하이퍼파라미터를 더 철저히 튜닝할 수 있어, 최종 모델이 더 나아질 가능성이 있습니다.
    -   개발 사이클의 지연 시간이 줄어들어 새로운 아이디어를 더 자주 테스트할 수 있습니다.
-   배치 크기를 늘리면 자원 소비가 줄어들 수도, 늘어날 수도, 변화가 없을 수도 있습니다.
-   배치 크기는 검증 세트 성능을 위해 튜닝해야 하는 하이퍼파라미터로 간주해서는 안 됩니다.
    -   모든 하이퍼파라미터가 잘 튜닝되고(특히 학습률과 정규화 하이퍼파라미터) 충분한 훈련 단계가 있다면, 동일한 최종 성능을 어떤 배치 크기에서도 달성할 수 있습니다. [Shallue et al. 2018](https://arxiv.org/abs/1811.03600)
    -   참고자료 [왜 배치 크기를 검증 세트 성능을 직접적으로 향상시키기 위해 튜닝해서는 안 되는가??](#why-shouldnt-the-batch-size-be-tuned-to-directly-improve-validation-set-performance)

#### 가능한 배치 크기 결정 및 훈련 처리량 추정


<details><summary><em>[Click to expand]</em></summary>

<br>

-   특정 모델과 최적화 알고리즘을 사용할 때, 하드웨어가 지원할 수 있는 배치 크기 범위가 있습니다. 이때 주로 제약이 되는 요소는 GPU 메모리입니다.
-   안타깝게도, 전체 훈련 프로그램을 실제로 실행하거나 컴파일해 보지 않으면 메모리에 맞는 배치 크기를 미리 계산하기 어렵습니다.
-   가장 쉬운 방법은 작은 단계 수로 훈련을 진행하면서, 배치 크기를 다르게 설정해보는 것입니다. 예를 들어, 배치 크기를 2의 배수로 늘리면서 실행해보다가 메모리 한계를 초과하는 지점을 찾는 방식입니다.
-   각 배치 크기에서 훈련 처리량을 신뢰할 수 있을 만큼 충분히 훈련해야 합니다

<p align="center">training 처리량 = (# 초당 처리된 데이터 수)</p>

<p align="center">or, equivalently, the <em>time per step</em>.</p>

<p align="center">time per step = (batch size) / (training 처리량)</p>

-   훈련 처리량은 말 그대로 모델이 주어진 시간 동안 처리할 수 있는 데이터 양을 의미합니다. 구체적으로는 초당 처리할 수 있는 샘플의 개수로 표현됩니다. 이 값이 높을수록 모델이 더 많은 데이터를 빠르게 학습할 수 있습니다.
-   배치 크기를 두배로 늘렸다면 동일한 시간동안 처리하는 데이터(훈련 처리량)가 대략 두배가 되어야 합니다.
-   만약 그렇지 않다면, 훈련 과정에 I/O 병목이나 계산 노드 간의 동기화 문제가 있을 수 있습니다. 이 문제는 훈련을 진행하기 전에 해결하는 것이 좋습니다.
-   만약 훈련 처리량이 특정 배치 크기까지 증가하다가 멈춘다면, 하드웨어가 더 큰 배치 크기를 지원하더라도 그 최대 배치 크기까지만 사용하는 것이 좋습니다.
    -   배치 크기를 크게 사용하는 이점은 훈련 처리량이 증가할 때만 얻을 수 있습니다. 그렇지 않다면 병목을 해결하거나 더 작은 배치 크기를 사용하는 것이 좋습니다.
    -   **Gradient accumulation** 하드웨어가 지원하는 것보다 더 큰 배치 크기를 시뮬레이션하는 것이지만, 실제 처리량 향상에는 도움이 되지 않으므로, 일반적으로 사용하지 않는 것이 좋습니다.
-   모델이나 최적화 알고리즘이 변경될 때마다 이 과정을 다시 반복해야 할 수도 있습니다(예: 다른 모델 구조에서는 더 큰 배치 크기가 가능할 수 있음).

</details>

#### Choosing the batch size to minimize training time (학습 시간을 최소화하기 위한 배치 크기 선택)

<details><summary><em>[Click to expand]</em></summary>

<br>

<p align="center">**학습 시간 = (단계당 시간) x (전체 단계 수)**</p>

   - 일반적으로 모든 가능한 배치 크기에 대해 단계당 시간은 거의 일정하다고 간주할 수 있습니다. 이는 병렬 연산에서의 오버헤드가 없고, 모든 학습 병목 현상이 이미 진단 및 해결된 경우에 해당합니다(학습 병목 현상의 진단 방법은 이전 섹션을 참조하세요). 하지만 실제로는 배치 크기가 증가하면 어느 정도 오버헤드가 발생하는 경우가 많습니다.
   -  (see the
    [previous section](#determining-the-feasible-batch-sizes-and-estimating-training-throughput)
   - 배치 크기가 커질수록 고정된 성능 목표에 도달하기 위해 필요한 전체 단계 수는 일반적으로 감소합니다(배치 크기가 변경될 때 관련 하이퍼파라미터가 재조정되는 경우; Shallue et al. 2018 참고). [Shallue et al. 2018](https://arxiv.org/abs/1811.03600)).
      - 예를 들어, 배치 크기를 두 배로 늘리면 전체 단계 수가 절반으로 줄어들 수 있습니다. 이를 **완벽한 스케일링(perfect scaling)** 이라고 합니다.
      - 완벽한 스케일링은 특정 임계 배치 크기까지 적용되며, 이 임계 크기를 넘으면 증가하는 배치 크기에도 불구하고 얻을 수 있는 이점이 점차 줄어듭니다.
      - 결국, 배치 크기를 계속 늘려도 학습 단계 수가 더 이상 줄어들지 않게 됩니다(단, 학습 단계 수가 증가하지는 않습니다).
따라서 학습 시간을 최소화하는 배치 크기는 학습 단계 수를 줄이는 효과가 있는 가장 큰 배치 크기입니다.
이 배치 크기는 데이터셋, 모델, 그리고 최적화 알고리즘에 따라 다르며, 이 배치 크기를 계산하는 방법은 아직 명확하지 않아, 새로운 문제마다 실험을 통해 찾아야 합니다. 🤖
      - 배치 크기를 비교할 때는 예제 예산(example budget)/에포크 예산([epoch](https://developers.google.com/machine-learning/glossary#epoch) 즉, 고정된 학습 데이터 수로 실험을 진행하는 방식)과 단계 예산(step budget, 고정된 학습 단계 수로 실험을 진행하는 방식)의 차이를 주의해야 합니다.
         - 에포크 예산으로 배치 크기를 비교하면, 큰 배치 크기가 학습 단계 수를 줄임으로써 유의미한 속도 향상을 제공할 수 있는 경우에도 완벽한 스케일링 영역만 탐색하게 됩니다.
      - 종종, 사용 가능한 하드웨어가 지원하는 가장 큰 배치 크기는 임계 배치 크기보다 작습니다. 따라서, 실험을 진행하지 않고 적용할 수 있는 좋은 규칙은 가능한 가장 큰 배치 크기를 사용하는 것입니다.
   - 만약 더 큰 배치 크기를 사용했는데 학습 시간이 오히려 증가한다면, 더 큰 배치 크기를 사용할 이유가 없습니다.

</details>

#### Choosing the batch size to minimize resource consumption

<details><summary><em>[Click to expand]</em></summary>

<br>


-   배치 크기를 증가시키는 데 관련된 두 가지 유형의 리소스 비용이 있습니다:
    1.  초기 비용: 새로운 하드웨어 구매 하거나 multi-GPU / multi-TPU 학습을 구현하기 위한 학습 파이프라인 재작성 할때 발생하는 비용.
    2.  사용 비용: 팀의 리소스 예산에 대한 청구, 클라우드 제공업체로부터의 청구, 전기/유지보수 비용 등.
-   배치 크기를 늘리는 데 상당한 초기 비용이 든다면, 프로젝트가 성숙해지고 비용-이익 트레이드오프를 더 쉽게 평가할 수 있을 때까지 배치 크기 증가를 미루는 것이 좋을 수 있습니다. multi-host 병렬 학습 프로그램을 구현하면 버그와 미묘한 문제가 발생할 수 있으므로, 처음에는 더 간단한 파이프라인으로 시작하는 것이 좋습니다. (반면, 학습 시간의 큰 단축은 많은 튜닝 실험이 필요한 초기 과정에서 매우 유익할 수 있습니다).
    [bugs](#considerations-for-multi-host-pipelines) and
    [subtle issues](#batch-normalization-implementation-details) 
-   우리는 총 사용 비용(여러 종류의 비용을 포함)을 "리소스 소비(resource consumption)"라고 부릅니다. 리소스 소비는 다음과 같이 나눌 수 있습니다:
  
<p align="center">Resource consumption = (resource consumption per step) x (total number of steps)</p>
<p align="center">리소스 소비 = (단계당 리소스 소비) x (총 단계 수)</p>


-   배치 크기를 늘리면 일반적으로 총 단계 수를 줄일 수 있습니다.[reduce the total number of steps](#choosing-the-batch-size-to-minimize-training-time). 리소스 소비가 증가하는지 감소하는지는 단계당 소비가 어떻게 변하는지에 따라 달라집니다.
    -   배치 크기 증가가 리소스 소비를 감소시킬 수 있습니다. 예를 들어, 더 큰 배치 크기로 각 단계를 같은 하드웨어에서 실행할 수 있다면(단계당 시간이 약간만 증가), 단계당 리소스 소비의 증가는 단계 수 감소에 의해 상쇄될 수 있습니다.
    -   배치 크기 증가가 리소스 소비를 변경하지 않을 수 있습니다. 예를 들어, 배치 크기를 두 배로 늘리면 필요한 단계 수가 절반으로 줄고 사용하는 GPU 수가 두 배가 되어, 총 소비(GPU-시간 측면에서)가 변하지 않을 수 있습니다.
    -   배치 크기 증가가 리소스 소비를 증가시킬 수 있습니다. 예를 들어, 배치 크기 증가에 업그레이드된 하드웨어가 필요하다면, 단계당 소비 증가가 단계 수 감소보다 더 커질 수 있습니다.
    -   
</details>

#### Changing the batch size requires re-tuning most hyperparameters

<details><summary><em>[Click to expand]</em></summary>

<br>

배치 크기를 변경하면 대부분의 하이퍼 파라미터를 다시 조정해야합니다.

-   The optimal values of most hyperparameters are sensitive to the batch size.
    Therefore, changing the batch size typically requires starting the tuning
    process all over again.
-   The hyperparameters that interact most strongly with the batch size, and therefore are most important to tune separately for each batch size, are the optimizer hyperparameters (e.g. learning rate, momentum) and the regularization hyperparameters.
-   Keep this in mind when choosing the batch size at the start of a project. If
    you need to switch to a different batch size later on, it might be
    difficult, time consuming, and expensive to re-tune everything for the new
    batch size.

</details>

#### How batch norm interacts with the batch size

<details><summary><em>[Click to expand]</em></summary>

<br>


-   Batch norm is complicated and, in general, should use a different batch size
    than the gradient computation to compute statistics. See the
    [batch norm section](#batch-normalization-implementation-details) for a
    detailed discussion.

</details>

### Choosing the initial configuration

-   Before beginning hyperparameter tuning we must determine the starting point.
    This includes specifying (1) the model configuration (e.g. number of
    layers), (2) the optimizer hyperparameters (e.g. learning rate), and (3) the
    number of training steps.
-   Determining this initial configuration will require some manually configured
    training runs and trial-and-error.
-   Our guiding principle is to find a simple, relatively fast, relatively
    low-resource-consumption configuration that obtains a "reasonable" result.
    -   "Simple" means avoiding bells and whistles wherever possible; these can
        always be added later. Even if bells and whistles prove helpful down the
        road, adding them in the initial configuration risks wasting time tuning
        unhelpful features and/or baking in unnecessary complications.
        -   For example, start with a constant learning rate before adding fancy
            decay schedules.
    -   Choosing an initial configuration that is fast and consumes minimal
        resources will make hyperparameter tuning much more efficient.
        -   For example, start with a smaller model.
    -   "Reasonable" performance depends on the problem, but at minimum means
        that the trained model performs much better than random chance on the
        validation set (although it might be bad enough to not be worth
        deploying).
-   Choosing the number of training steps involves balancing the following
    tension:
    -   On the one hand, training for more steps can improve performance and
        makes hyperparameter tuning easier (see
        [Shallue et al. 2018](https://arxiv.org/abs/1811.03600)).
    -   On the other hand, training for fewer steps means that each training run
        is faster and uses fewer resources, boosting tuning efficiency by
        reducing the time between cycles and allowing more experiments to be run
        in parallel. Moreover, if an unnecessarily large step budget is chosen
        initially, it might be hard to change it down the road, e.g. once the
        learning rate schedule is tuned for that number of steps.

## A scientific approach to improving model performance

For the purposes of this document, the ultimate goal of machine learning
development is to maximize the utility of the deployed model. Even though many
aspects of the development process differ between applications (e.g. length of
time, available computing resources, type of model), we can typically use the
same basic steps and principles on any problem.

Our guidance below makes the following assumptions:

-   There is already a fully-running training pipeline along with a
    configuration that obtains a reasonable result.
-   There are enough computational resources available to conduct meaningful
    tuning experiments and run at least several training jobs in parallel.

### The incremental tuning strategy

***Summary:*** *Start with a simple configuration and incrementally make
improvements while building up insight into the problem. Make sure that any
improvement is based on strong evidence to avoid adding unnecessary complexity.*

-   Our ultimate goal is to find a configuration that maximizes the performance
    of our model.
    -   In some cases, our goal will be to maximize how much we can improve the
        model by a fixed deadline (e.g. submitting to a competition).
    -   In other cases, we want to keep improving the model indefinitely (e.g.
        continually improving a model used in production).
-   In principle, we could maximize performance by using an algorithm to
    automatically search the entire space of possible configurations, but this
    is not a practical option.
    -   The space of possible configurations is extremely large and there are
        not yet any algorithms sophisticated enough to efficiently search this
        space without human guidance.
-   Most automated search algorithms rely on a hand-designed *search space* that
    defines the set of configurations to search in, and these search spaces can
    matter quite a bit.
-   The most effective way to maximize performance is to start with a simple
    configuration and incrementally add features and make improvements while
    building up insight into the problem.
    -   We use automated search algorithms in each round of tuning and
        continually update our search spaces as our understanding grows.
-   As we explore, we will naturally find better and better configurations and
    therefore our "best" model will continually improve.
    -   We call it a *launch* when we update our best configuration (which may
        or may not correspond to an actual launch of a production model).
    -   For each launch, we must make sure that the change is based on strong
        evidence – not just random chance based on a lucky configuration – so
        that we don't add unnecessary complexity to the training pipeline.

At a high level, our incremental tuning strategy involves repeating the
following four steps:

1.  Identify an appropriately-scoped goal for the next round of experiments.
2.  Design and run a set of experiments that makes progress towards this goal.
3.  Learn what we can from the results.
4.  Consider whether to launch the new best configuration.

The remainder of this section will consider this strategy in much greater
detail.

### Exploration vs exploitation

***Summary:*** *Most of the time, our primary goal is to gain insight into the
problem.*

-   Although one might think we would spend most of our time trying to maximize
    performance on the validation set, in practice we spend the majority of our
    time trying to gain insight into the problem, and comparatively little time
    greedily focused on the validation error.
    -   In other words, we spend most of our time on "exploration" and only a
        small amount on "exploitation".
-   In the long run, understanding the problem is critical if we want to
    maximize our final performance. Prioritizing insight over short term gains
    can help us:
    -   Avoid launching unnecessary changes that happened to be present in
        well-performing runs merely through historical accident.
    -   Identify which hyperparameters the validation error is most sensitive
        to, which hyperparameters interact the most and therefore need to be
        re-tuned together, and which hyperparameters are relatively insensitive
        to other changes and can therefore be fixed in future experiments.
    -   Suggest potential new features to try, such as new regularizers if
        overfitting is an issue.
    -   Identify features that don't help and therefore can be removed, reducing
        the complexity of future experiments.
    -   Recognize when improvements from hyperparameter tuning have likely
        saturated.
    -   Narrow our search spaces around the optimal value to improve tuning
        efficiency.
-   When we are eventually ready to be greedy, we can focus purely on the
    validation error even if the experiments aren't maximally informative about
    the structure of the tuning problem.

### Choosing the goal for the next round of experiments

***Summary:*** *Each round of experiments should have a clear goal and be
sufficiently narrow in scope that the experiments can actually make progress
towards the goal.*

-   Each round of experiments should have a clear goal and be sufficiently
    narrow in scope that the experiments can actually make progress towards the
    goal: if we try to add multiple features or answer multiple questions at
    once, we may not be able to disentangle the separate effects on the results.
-   Example goals include:
    -   Try a potential improvement to the pipeline (e.g. a new regularizer,
        preprocessing choice, etc.).
    -   Understand the impact of a particular model hyperparameter (e.g. the
        activation function)
    -   Greedily minimize validation error.

### Designing the next round of experiments

***Summary:*** *Identify which hyperparameters are scientific, nuisance, and
fixed hyperparameters for the experimental goal. Create a sequence of studies to
compare different values of the scientific hyperparameters while optimizing over
the nuisance hyperparameters. Choose the search space of nuisance
hyperparameters to balance resource costs with scientific value.*

#### Identifying scientific, nuisance, and fixed hyperparameters

<details><summary><em>[Click to expand]</em></summary>

<br>

-   For a given goal, all hyperparameters will be either **scientific
    hyperparameters**, **nuisance hyperparameters**, or **fixed
    hyperparameters**.
    -   Scientific hyperparameters are those whose effect on the model's
        performance we're trying to measure.
    -   Nuisance hyperparameters are those that need to be optimized over in
        order to fairly compare different values of the scientific
        hyperparameters. This is similar to the statistical concept of
        [nuisance parameters](https://en.wikipedia.org/wiki/Nuisance_parameter).
    -   Fixed hyperparameters will have their values fixed in the current round
        of experiments. These are hyperparameters whose values do not need to
        (or we do not want them to) change when comparing different values of
        the scientific hyperparameters.
        -   By fixing certain hyperparameters for a set of experiments, we must
            accept that conclusions derived from the experiments might not be
            valid for other settings of the fixed hyperparameters. In other
            words, fixed hyperparameters create caveats for any conclusions we
            draw from the experiments.
-   For example, if our goal is to "determine whether a model with more hidden
    layers will reduce validation error", then the number of hidden layers is a
    scientific hyperparameter.
    -   The learning rate is a nuisance hyperparameter because we can only
        fairly compare models with different numbers of hidden layers if the
        learning rate is tuned separately for each number of layers (the optimal
        learning rate generally depends on the model architecture).
    -   The activation function could be a fixed hyperparameter if we have
        determined in prior experiments that the best choice of activation
        function is not sensitive to model depth, or if we are willing to limit
        our conclusions about the number of hidden layers to only cover this
        specific choice of activation function. Alternatively, it could be a
        nuisance parameter if we are prepared to tune it separately for each
        number of hidden layers.
-   Whether a particular hyperparameter is a scientific hyperparameter, nuisance
    hyperparameter, or fixed hyperparameter is not inherent to that
    hyperparameter, but changes depending on the experimental goal.
    -   For example, the choice of activation function could be a scientific
        hyperparameter (is ReLU or tanh a better choice for our problem?), a
        nuisance hyperparameter (is the best 5-layer model better than the best
        6-layer model when we allow several different possible activation
        functions?), or a fixed hyperparameter (for ReLU nets, does adding batch
        normalization in a particular position help?).
-   When designing a new round of experiments, we first identify the scientific
    hyperparameters for our experimental goal.
    -   At this stage, we consider all other hyperparameters to be nuisance
        hyperparameters.
-   Next, we convert some of the nuisance hyperparameters into fixed
    hyperparameters.
    -   With limitless resources, we would leave all non-scientific
        hyperparameters as nuisance hyperparameters so that the conclusions we
        draw from our experiments are free from caveats about fixed
        hyperparameter values.
    -   However, the more nuisance hyperparameters we attempt to tune, the
        greater the risk we fail to tune them sufficiently well for each setting
        of the scientific hyperparameters and end up reaching the wrong
        conclusions from our experiments.
        -   As described
            [below](#striking-a-balance-between-informative-and-affordable-experiments),
            we could counter this risk by increasing the computational budget,
            but often our maximum resource budget is less than would be needed
            to tune over all non-scientific hyperparameters.
    -   We choose to convert a nuisance hyperparameter into a fixed
        hyperparameter when, in our judgment, the caveats introduced by fixing
        it are less burdensome than the cost of including it as a nuisance
        hyperparameter.
        -   The more a given nuisance hyperparameter interacts with the
            scientific hyperparameters, the more damaging it is to fix its
            value. For example, the best value of the weight decay strength
            typically depends on the model size, so comparing different model
            sizes assuming a single specific value of the weight decay would not
            be very insightful.
-   Although the type we assign to each hyperparameter depends on the
    experimental goal, we have the following rules of thumb for certain
    categories of hyperparameters:
    -   Of the various optimizer hyperparameters (e.g. the learning rate,
        momentum, learning rate schedule parameters, Adam betas etc.), at least
        some of them will be nuisance hyperparameters because they tend to
        interact the most with other changes.
        -   They are rarely scientific hyperparameters because a goal like "what
            is the best learning rate for the current pipeline?" doesn't give
            much insight – the best setting could easily change with the next
            pipeline change anyway.
        -   Although we might fix some of them occasionally due to resource
            constraints or when we have particularly strong evidence that they
            don't interact with the scientific parameters, we should generally
            assume that optimizer hyperparameters must be tuned separately to
            make fair comparisons between different settings of the scientific
            hyperparameters, and thus shouldn't be fixed.
            -   Furthermore, we have no *a priori* reason to prefer one
                optimizer hyperparameter value over another (e.g. they don't
                usually affect the computational cost of forward passes or
                gradients in any way).
    -   In contrast, the *choice* of optimizer is typically a scientific
        hyperparameter or fixed hyperparameter.
        -   It is a scientific hyperparameter if our experimental goal involves
            making fair comparisons between two or more different optimizers
            (e.g. "determine which optimizer produces the lowest validation
            error in a given number of steps").
        -   Alternatively, we might make it a fixed hyperparameter for a variety
            of reasons, including (1) prior experiments make us believe that the
            best optimizer for our problem is not sensitive to current
            scientific hyperparameters; and/or (2) we prefer to compare values
            of the scientific hyperparameters using this optimizer because its
            training curves are easier to reason about; and/or (3) we prefer to
            use this optimizer because it uses less memory than the
            alternatives.
    -   Hyperparameters introduced by a regularization technique are typically
        nuisance hyperparameters, but whether or not we include the
        regularization technique at all is a scientific or fixed hyperparameter.
        -   For example, dropout adds code complexity, so when deciding whether
            to include it we would make "no dropout" vs "dropout" a scientific
            hyperparameter and the dropout rate a nuisance hyperparameter.
            -   If we decide to add dropout to our pipeline based on this
                experiment, then the dropout rate would be a nuisance
                hyperparameter in future experiments.
    -   Architectural hyperparameters are often scientific or fixed
        hyperparameters because architecture changes can affect serving and
        training costs, latency, and memory requirements.
        -   For example, the number of layers is typically a scientific or fixed
            hyperparameter since it tends to have dramatic consequences for
            training speed and memory usage.
-   In some cases, the sets of nuisance and fixed hyperparameters will depend on
    the values of the scientific hyperparameters.
    -   For example, suppose we are trying to determine which optimizer out of
        Nesterov momentum and Adam results in the lowest validation error. The
        scientific hyperparameter is the `optimizer`, which takes values
        `{"Nesterov_momentum", "Adam"}`. The value
        `optimizer="Nesterov_momentum"` introduces the nuisance/fixed
        hyperparameters `{learning_rate, momentum}`, but the value
        `optimizer="Adam"` introduces the nuisance/fixed hyperparameters
        `{learning_rate, beta1, beta2, epsilon}`.
    -   Hyperparameters that are only present for certain values of the
        scientific hyperparameters are called **conditional hyperparameters**.
    -   We should not assume two conditional hyperparameters are the same just
        because they have the same name! In the above example, the conditional
        hyperparameter called `learning_rate` is a *different* hyperparameter
        for `optimizer="Nesterov_momentum"` versus `optimizer="Adam"`. Its role
        is similar (although not identical) in the two algorithms, but the range
        of values that work well in each of the optimizers is typically
        different by several orders of magnitude.

</details>

#### Creating a set of studies

<details><summary><em>[Click to expand]</em></summary>

<br>


-   Once we have identified the scientific and nuisance hyperparameters, we
    design a "study" or sequence of studies to make progress towards the
    experimental goal.
    -   A study specifies a set of hyperparameter configurations to be run for
        subsequent analysis. Each configuration is called a "trial".
    -   Creating a study typically involves choosing the hyperparameters that
        will vary across trials, choosing what values those hyperparameters can
        take on (the "search space"), choosing the number of trials, and
        choosing an automated search algorithm to sample that many trials from
        the search space. Alternatively, we could create a study by specifying
        the set of hyperparameter configurations manually.
-   The purpose of the studies is to run the pipeline with different values of
    the scientific hyperparameters, while at the same time **"optimizing away"**
    (or "optimizing over") the nuisance hyperparameters so that comparisons
    between different values of the scientific hyperparameters are as fair as
    possible.
-   In the simplest case, we would make a separate study for each configuration
    of the scientific parameters, where each study tunes over the nuisance
    hyperparameters.
    -   For example, if our goal is to select the best optimizer out of Nesterov
        momentum and Adam, we could create one study in which
        `optimizer="Nesterov_momentum"` and the nuisance hyperparameters are
        `{learning_rate, momentum}`, and another study in which
        `optimizer="Adam"` and the nuisance hyperparameters are `{learning_rate,
        beta1, beta2, epsilon}`. We would compare the two optimizers by
        selecting the best performing trial from each study.
    -   We can use any gradient-free optimization algorithm, including methods
        such as Bayesian optimization or evolutionary algorithms, to optimize
        over the nuisance hyperparameters, although
        [we prefer](#why-use-quasi-random-search-instead-of-more-sophisticated-black-box-optimization-algorithms-during-the-exploration-phase-of-tuning)
        to use quasi-random search in the
        [exploration phase](#exploration-vs-exploitation) of tuning because of a
        variety of advantages it has in this setting.
        [After exploration concludes](#after-exploration-concludes), if
        state-of-the-art Bayesian optimization software is available, that is
        our preferred choice.
-   In the more complicated case where we want to compare a large number of
    values of the scientific hyperparameters and it is impractical to make that
    many independent studies, we can include the scientific parameters in the
    same search space as the nuisance hyperparameters and use a search algorithm
    to sample values of *both* the scientific and nuisance hyperparameters in a
    single study.
    -   When taking this approach, conditional hyperparameters can cause
        problems since it is hard to specify a search space unless the set of
        nuisance hyperparameters is the same for all values of the scientific
        hyperparameters.
    -   In this case,
        [our preference](#why-use-quasi-random-search-instead-of-more-sophisticated-black-box-optimization-algorithms-during-the-exploration-phase-of-tuning)
        for using quasi-random search over fancier black-box optimization tools
        is even stronger, since it ensures that we obtain a relatively uniform
        sampling of values of the scientific hyperparameters. Regardless of the
        search algorithm, we need to make sure somehow that it searches the
        scientific parameters uniformly.

</details>

#### Striking a balance between informative and affordable experiments

<details><summary><em>[Click to expand]</em></summary>

<br>


-   When designing a study or sequence of studies, we need to allocate a limited
    budget in order to adequately achieve the following three desiderata:
    1.  Comparing enough different values of the scientific hyperparameters.
    2.  Tuning the nuisance hyperparameters over a large enough search space.
    3.  Sampling the search space of nuisance hyperparameters densely enough.
-   The better we can achieve these three desiderata, the more insight we can
    extract from our experiment.
    -   Comparing as many values of the scientific hyperparameters as possible
        broadens the scope of the insights we gain from the experiment.
    -   Including as many nuisance hyperparameters as possible and allowing each
        nuisance hyperparameter to vary over as wide a range as possible
        increases our confidence that a "good" value of the nuisance
        hyperparameters **exists** in the search space for each configuration of
        the scientific hyperparameters.
        -   Otherwise, we might make unfair comparisons between values of the
            scientific hyperparameters by not searching possible regions of the
            nuisance parameter space where better values might lie for some
            values of the scientific parameters.
    -   Sampling the search space of nuisance hyperparameters as densely as
        possible increases our confidence that any good settings for the
        nuisance hyperparameters that happen to exist in our search space will
        be found by the search procedure.
        -   Otherwise, we might make unfair comparisons between values of the
            scientific parameters due to some values getting luckier with the
            sampling of the nuisance hyperparameters.
-   Unfortunately, improvements in *any* of these three dimensions require
    either increasing the number of trials, and therefore increasing the
    resource cost, or finding a way to save resources in one of the other
    dimensions.
    -   Every problem has its own idiosyncrasies and computational constraints,
        so how to allocate resources across these three desiderata requires some
        level of domain knowledge.
    -   After running a study, we always try to get a sense of whether the study
        tuned the nuisance hyperparameters well enough (i.e. searched a large
        enough space extensively enough) to fairly compare the scientific
        hyperparameters (as described in greater detail
        [below](#extracting-insight-from-experimental-results)).

</details>

### Extracting insight from experimental results

***Summary:*** *In addition to trying to achieve the original scientific goal of
each group of experiments, go through a checklist of additional questions and,
if issues are discovered, revise the experiments and rerun them.*

-   Ultimately, each group of experiments has a specific goal and we want to
    evaluate the evidence the experiments provide toward that goal.
    -   However, if we ask the right questions, we will often find issues that
        need to be corrected before a given set of experiments can make much
        progress towards their original goal.
        -   If we don’t ask these questions, we may draw incorrect conclusions.
    -   Since running experiments can be expensive, we also want to take the
        opportunity to extract other useful insights from each group of
        experiments, even if these insights are not immediately relevant to the
        current goal.
-   Before analyzing a given set of experiments to make progress toward their
    original goal, we should ask ourselves the following additional questions:
    -   [Is the search space large enough?](#identifying-bad-search-space-boundaries)
        -   If the optimal point from a study is near the boundary of the search
            space in one or more dimensions, the search is probably not wide
            enough. In this case, we should run another study with an expanded
            search space.
    -   [Have we sampled enough points from the search space?](#not-sampling-enough-points-in-the-search-space)
        -   If not, run more points or be less ambitious in the tuning goals.
    -   What fraction of the trials in each study are **infeasible** (i.e.
        trials that diverge, get really bad loss values, or fail to run at all
        because they violate some implicit constraint)?
        -   When a very large fraction of points in a study are **infeasible**
            we should try to adjust the search space to avoid sampling such
            points, which sometimes requires reparameterizing the search space.
        -   In some cases, a large number of infeasible points can indicate a
            bug in the training code.
    -   [Does the model exhibit optimization issues?](#how-can-optimization-failures-be-debugged-and-mitigated)
    -   [What can we learn from the training curves of the best trials?](#examining-the-training-curves)
        -   For example, do the best trials have training curves consistent with
            problematic overfitting?
-   If necessary, based on the answers to the questions above, refine the most
    recent study (or group of studies) to improve the search space and/or sample
    more trials, or take some other corrective action.
-   Once we have answered the above questions, we can move on to evaluating the
    evidence the experiments provide towards our original goal (for example,
    [evaluating whether a change is useful](#detecting-whether-a-change-is-useful-with-isolation-plots)).

#### Identifying bad search space boundaries

<details><summary><em>[Click to expand]</em></summary>

<br>


-   A search space is suspicious if the best point sampled from it is close to
    its boundary. We might find an even better point if we expanded the search
    range in that direction.
-   To check search space boundaries, we like to plot completed trials on what
    we call **basic hyperparameter axis plots** where we plot the validation
    objective value versus one of the hyperparameters (e.g. learning rate). Each
    point on the plot corresponds to a single trial.
    -   The validation objective value for each trial should usually be the best
        value it achieved over the course of training.

<p align="center" id="figure-1">
<img src="assets/good_and_bad_search_spaces.png" width="98%" alt="Example of good search space boundaries">
</p>

<p align="center"><b>Figure 1:</b> Examples of bad search space boundaries and acceptable search space boundaries.</p>

-   The plots in [Figure 1](#figure-1) show the error rate (lower is better)
    against the initial learning rate.
-   If the best points cluster towards the edge of a search space (in some
    dimension), then the search space boundaries might need to be expanded until
    the best observed point is no longer close to the boundary.
-   Often, a study will include "infeasible" trials that diverge or get very bad
    results (marked with red Xs in the above plots).
    -   If all trials are infeasible for learning rates greater than some
        threshold value, and if the best performing trials have learning rates
        at the edge of that region, the model [may suffer from stability issues
        preventing it from accessing higher learning
        rates](#how-can-optimization-failures-be-debugged-and-mitigated).

</details>

#### Not sampling enough points in the search space

<details><summary><em>[Click to expand]</em></summary>

<br>


-   In general,
    [it can be very difficult to know](#how-many-trials-are-needed-to-get-good-results-with-quasi-random-search)
    if the search space has been sampled densely enough. 🤖
-   Running more trials is of course better, but comes at an obvious cost.
-   Since it is so hard to know when we have sampled enough, we usually sample
    what we can afford and try to calibrate our intuitive confidence from
    repeatedly looking at various hyperparameter axis plots and trying to get a
    sense of how many points are in the "good" region of the search space.

</details>

#### Examining the training curves

<details><summary><em>[Click to expand]</em></summary>

<br>


***Summary:*** *Examining the training curves is an easy way to identify common
failure modes and can help us prioritize what actions to take next.*

-   Although in many cases the primary objective of our experiments only
    requires considering the validation error of each trial, we must be careful
    when reducing each trial to a single number because it can hide important
    details about what’s going on below the surface.
-   For every study, we always look at the **training curves** (training error
    and validation error plotted versus training step over the duration of
    training) of at least the best few trials.
-   Even if this is not necessary for addressing the primary experimental
    objective, examining the training curves is an easy way to identify common
    failure modes and can help us prioritize what actions to take next.
-   When examining the training curves, we are interested in the following
    questions.
-   Are any of the trials exhibiting **problematic overfitting?**
    -   Problematic overfitting occurs when the validation error starts
        *increasing* at some point during training.
    -   In experimental settings where we optimize away nuisance hyperparameters
        by selecting the "best" trial for each setting of the scientific
        hyperparameters, we should check for problematic overfitting in *at
        least* each of the best trials corresponding to the settings of the
        scientific hyperparameters that we’re comparing.
        -   If any of the best trials exhibits problematic overfitting, we
            usually want to re-run the experiment with additional regularization
            techniques and/or better tune the existing regularization parameters
            before comparing the values of the scientific hyperparameters.
            -   This may not apply if the scientific hyperparameters include
                regularization parameters, since then it would not be surprising
                if low-strength settings of those regularization parameters
                resulted in problematic overfitting.
        -   Reducing overfitting is often straightforward using common
            regularization techniques that add minimal code complexity or extra
            computation (e.g. dropout, label smoothing, weight decay), so it’s
            usually no big deal to add one or more of these to the next round of
            experiments.
        -   For example, if the scientific hyperparameter is "number of hidden
            layers" and the best trial that uses the largest number of hidden
            layers exhibited problematic overfitting, then we would usually
            prefer to try it again with additional regularization instead of
            immediately selecting the smaller number of hidden layers.
        -   Even if none of the "best" trials are exhibiting problematic
            overfitting, there might still be a problem if it occurs in *any* of
            the trials.
            -   Selecting the best trial suppresses configurations exhibiting
                problematic overfitting and favors those that do not. In other
                words, it will favor configurations with more regularization.
            -   However, anything that makes training worse can act as a
                regularizer, even if it wasn't intended that way. For example,
                choosing a smaller learning rate can regularize training by
                hobbling the optimization process, but we typically don't want
                to choose the learning rate this way.
            -   So we must be aware that the "best" trial for each setting of
                the scientific hyperparameters might be selected in such a way
                that favors "bad" values of some of the scientific or nuisance
                hyperparameters.
-   Is there high step-to-step variance in the training or validation error late
    in training?
    -   If so, this could interfere with our ability to compare different values
        of the scientific hyperparameters (since each trial randomly ends on a
        "lucky" or "unlucky" step) and our ability to reproduce the result of
        the best trial in production (since the production model might not end
        on the same "lucky" step as in the study).
    -   The most likely causes of step-to-step variance are batch variance (from
        randomly sampling examples from the training set for each batch), small
        validation sets, and using a learning rate that’s too high late in
        training.
    -   Possible remedies include increasing the batch size, obtaining more
        validation data, using learning rate decay, or using Polyak averaging.
-   Are the trials still improving at the end of training?
    -   If so, this indicates that we are in the
        ["compute bound" regime](#determining-the-number-of-steps-for-each-training-run)
        and we may benefit from
        [increasing the number of training steps](#Deciding-how-long-to-train-when-training-is-compute-bound)
        or changing the learning rate schedule.
-   Has performance on the training and validation sets saturated long before
    the final training step?
    -   If so, this indicates that we are in the
        ["not compute-bound"](#determining-the-number-of-steps-for-each-training-run)
        regime and that we may be able to
        [decrease the number of training steps](#deciding-how-long-to-train-when-training-is-not-compute-bound).
-   Although we cannot enumerate them all, there are many other additional
    behaviors that can become evident from examining the training curves (e.g.
    training loss *increasing* during training usually indicates a bug in the
    training pipeline).

</details>

#### Detecting whether a change is useful with isolation plots

<details><summary><em>[Click to expand]</em></summary>

<br>


<p align="center" id="figure-2">
<img src="assets/basic_isolation_plot.png" width="55%" alt="Isolation plot that investigates the best value of weight decay for ResNet-50
trained on ImageNet.">
</p>

<p align="center"><b>Figure 2:</b> Isolation plot that investigates the best value of weight decay for ResNet-50 trained on ImageNet.</p>

-   Often, the goal of a set of experiments is to compare different values of a
    scientific hyperparameter.
    -   For example, we may want to determine the value of weight decay that
        results in the best validation error.
-   An **isolation plot** is a special case of the basic hyperparameter axis
    plot. Each point on an isolation plot corresponds to the performance of the
    *best* trial across some (or all) of the nuisance hyperparameters.
    -   In other words, we plot the model performance after "optimizing away"
        the nuisance hyperparameters.
-   An isolation plot makes it easier to perform an apples-to-apples comparison
    between different values of the scientific hyperparameter.
-   For example, [Figure 2](#figure-2) reveals the value of weight decay that
    produces the best validation performance for a particular configuration of
    ResNet-50 trained on ImageNet.
    -   If our goal is to determine whether to include weight decay at all, then
        we would compare the best point from this plot against the baseline of
        no weight decay. For a fair comparison, the baseline should also have
        its learning rate equally well tuned.
-   When we have data generated by (quasi)random search and are considering a
    continuous hyperparameter for an isolation plot, we can approximate the
    isolation plot by bucketing the x-axis values of the basic hyperparameter
    axis plot and taking the best trial in each vertical slice defined by the
    buckets.

</details>

#### Automate generically useful plots

<details><summary><em>[Click to expand]</em></summary>

<br>

-   The more effort it is to generate plots, the less likely we are to look at
    them as much as we should, so it behooves us to set up our infrastructure to
    automatically produce as many of them as possible.
-   At a minimum, we automatically generate basic hyperparameter axis plots for
    all hyperparameters that we vary in an experiment.
-   Additionally, we automatically produce training curves for all trials and
    make it as easy as possible to find the best few trials of each study and
    examine their training curves.
-   There are many other potential plots and visualizations we can add that can
    be useful. Although the ones described above are a good starting point, to
    paraphrase Geoffrey Hinton, "Every time you plot something new, you learn
    something new."

</details>

### Determining whether to adopt a training pipeline change or hyperparameter configuration

***Summary:*** *When deciding whether to make a change to our model or training
procedure or adopt a new hyperparameter configuration going forward, we need to
be aware of the different sources of variation in our results.*

-   When we are trying to improve our model, we might observe that a particular
    candidate change initially achieves a better validation error compared to
    our incumbent configuration, but find that after repeating the experiment
    there is no consistent advantage. Informally, we can group the most
    important sources of variation that might cause such an inconsistent result
    into the following broad categories:
    -   **Training procedure variance**, **retrain variance**, or **trial
        variance**: the variation we see between training runs that use the same
        hyperparameters, but different random seeds.
        -   For example, different random initializations, training data
            shuffles, dropout masks, patterns of data augmentation operations,
            and orderings of parallel arithmetic operations, are all potential
            sources of trial variance.
    -   **Hyperparameter search variance**, or **study variance**: the variation
        in results caused by our procedure to select the hyperparameters.
        -   For example, we might run the same experiment with a particular
            search space, but with two different seeds for quasi-random search
            and end up selecting different hyperparameter values.
    -   **Data collection and sampling variance**: the variance from any sort of
        random split into training, validation, and test data or variance due to
        the training data generation process more generally.
-   It is all well and good to make comparisons of validation error rates
    estimated on a finite validation set using fastidious statistical tests, but
    often the trial variance alone can produce statistically significant
    differences between two different trained models that use the same
    hyperparameter settings.
-   We are most concerned about study variance when trying to make conclusions
    that go beyond the level of an individual point in hyperparameters space.
    -   The study variance depends on the number of trials and the search space
        and we have seen cases where it is larger than the trial variance as
        well as cases where it is much smaller.
-   Therefore, before adopting a candidate change, consider running the best
    trial N times to characterize the run-to-run trial variance.
    -   Usually, we can get away with only recharacterizing the trial variance
        after major changes to the pipeline, but in some applications we might
        need fresher estimates.
    -   In other applications, characterizing the trial variance is too costly
        to be worth it.
-   At the end of the day, although we only want to adopt changes (including new
    hyperparameter configurations) that produce real improvements, demanding
    complete certainty that something helps isn't the right answer either.
-   Therefore, if a new hyperparameter point (or other change) gets a better
    result than the baseline (taking into account the retrain variance of both
    the new point and the baseline as best we can), then we probably should
    adopt it as the new baseline for future comparisons.
    -   However, we should only adopt changes that produce improvements that
        outweigh any complexity they add.

### After exploration concludes

***Summary:*** *Bayesian optimization tools are a compelling option once we’re
done exploring for good search spaces and have decided what hyperparameters even
should be tuned at all.*

-   At some point, our priorities will shift from learning more about the tuning
    problem to producing a single best configuration to launch or otherwise use.
-   At this point, there should be a refined search space that comfortably
    contains the local region around the best observed trial and has been
    adequately sampled.
-   Our exploration work should have revealed the most essential hyperparameters
    to tune (as well as sensible ranges for them) that we can use to construct a
    search space for a final automated tuning study using as large a tuning
    budget as possible.
-   Since we no longer care about maximizing our insight into the tuning
    problem, many of
    [the advantages of quasi-random search](#why-use-quasi-random-search-instead-of-more-sophisticated-black-box-optimization-algorithms-during-the-exploration-phase-of-tuning)
    no longer apply and Bayesian optimization tools should be used to
    automatically find the best hyperparameter configuration.
    -   [Open-Source Vizier](https://github.com/google/vizier) implements
        a variety of sophisticated algorithms for tuning ML models, including
        Bayesian Optimization algorithms.
    -   If the search space contains a non-trivial volume of divergent points
        (points that get NaN training loss or even training loss many standard
        deviations worse than the mean), it is important to use black box
        optimization tools that properly handle trials that diverge (see
        [Bayesian Optimization with Unknown Constraints](https://arxiv.org/abs/1403.5607)
        for an excellent way to deal with this issue). [Open-Source Vizier](https://github.com/google/vizier)
        has support for divergent points by marking trials as infeasible, although it may not use our preferred approach from [Gelbart et al.](https://arxiv.org/abs/1403.5607), depending on how it is configured.
-   At this point, we should also consider checking the performance on the test
    set.
    -   In principle, we could even fold the validation set into the training
        set and retraining the best configuration found with Bayesian
        optimization. However, this is only appropriate if there won't be future
        launches with this specific workload (e.g. a one-time Kaggle
        competition).

## Determining the number of steps for each training run

-   There are two types of workloads: those that are compute-bound and those
    that are not.
-   When training is **compute-bound**, training is limited by how long we are
    willing to wait and not by how much training data we have or some other
    factor.
    -   In this case, if we can somehow train longer or more efficiently, we
        should see a lower training loss and, with proper tuning, an improved
        validation loss.
    -   In other words, *speeding up* training is equivalent to *improving*
        training and the "optimal" training time is always "as long as we can
        afford."
    -   That said, just because a workload is compute-limited doesn't mean
        training longer/faster is the only way to improve results.
-   When training is **not compute-bound**, we can afford to train as long as we
    would like to, and, at some point, training longer doesn't help much (or
    even causes problematic overfitting).
    -   In this case, we should expect to be able to train to very low training
        loss, to the point where training longer might slightly reduce the
        training loss, but will not meaningfully reduce the validation loss.
    -   Particularly when training is not compute-bound, a more generous
        training time budget can make tuning easier, especially when tuning
        learning rate decay schedules, since they have a particularly strong
        interaction with the training budget.
        -   In other words, very stingy training time budgets might require a
            learning rate decay schedule tuned to perfection in order to achieve
            a good error rate.
-   Regardless of whether a given workload is compute-bound or not, methods that
    increase the variance of the gradients (across batches) will usually result
    in slower training progress, and thus may increase the number of training
    steps required to reach a particular validation loss. High gradient variance
    can be caused by:
    -   Using a smaller batch size
    -   Adding data augmentation
    -   Adding some types of regularization (e.g. dropout)

### Deciding how long to train when training is *not* compute-bound

-   Our main goal is to ensure we are training long enough for the model to
    reach the best possible result, while avoiding being overly wasteful in the
    number of training steps.
-   When in doubt, err on the side of training longer. Performance should never
    degrade when training longer, assuming retrospective (optimal) checkpoint
    selection is used properly and checkpoints are frequent enough.
-   Never tune the `max_train_steps` number in a study. Pick a value and use it
    for all trials. From these trials, plot the training step that retrospective
    checkpoint selection finds in order to refine the choice of
    `max_train_steps`.
    -   For example, if the best step is always during the first 10% of
        training, then the maximum number of steps is way too high.
    -   Alternatively, if the best step is consistently in the last 25% of
        training we might benefit from training longer and re-tuning the decay
        schedule.
-   The ideal number of training steps can change when the architecture or data
    changes (e.g. adding data augmentation).
-   Below we describe how to pick an initial candidate value for
    `max_train_steps` based on the number of steps necessary to "perfectly fit"
    the training set using a constant learning rate.
    -   Note, we are not using the phrase "perfectly fit the training set" in a
        precise or mathematically well-defined way. It is merely meant as an
        informal descriptor to indicate a very low training loss.
        -   For example, when training with the log loss, absent regularization
            terms, we might see the training loss keep slowly improving until we
            reach floating point limits as the network weights grow without
            bound and the predictions of the model on the training set become
            increasingly confident. In this case, we might say the model
            "perfectly fit" the training set around the time the
            misclassification error reached zero on the training set.
    -   The starting value for `max_train_steps` we find may need to be
        increased if the amount of gradient noise in the training procedure
        increases.
        -   For example, if data augmentation or regularizers like dropout are
            introduced to the model.
    -   It may be possible to decrease `max_train_steps` if the training process
        improves somehow.
        -   For example, with a better tuned optimizer or a better tuned
            learning rate schedule.

#### Algorithm for picking an initial candidate for max_train_steps using a learning rate sweep

<details><summary><em>[Click to expand]</em></summary>

<br>

-   This procedure assumes it is possible to not only "perfectly" fit the
    training set, but to do so using a constant learning rate schedule.
-   If it is possible to perfectly fit the entire training set, then there must
    exist a configuration (with some value of `max_train_steps`) that perfectly
    fits the training set; find any such configuration and use its value of
    `max_train_steps` as a starting point `N`.
-   Run a constant learning rate sweep (i.e. grid search the learning rate)
    without data augmentation and without regularization where each trial trains
    for `N` steps.
-   The number of steps required for the fastest trial in the sweep to reach
    perfect training performance is our initial guess for `max_train_steps`.
-   **NOTE:** Bad search spaces can make it possible to engage in
    self-deception.
    -   For example, if all the learning rates in a study are too small, we
        might incorrectly conclude that a very large value of `max_train_steps`
        is necessary.
    -   At a minimum, we should check that the optimal learning rate in the
        study is not at the boundary of the search space.

</details>

### Deciding how long to train when training is compute-bound

-   In some cases, training loss keeps improving indefinitely and our patience
    and computational resources become the limiting factors.
-   If training loss (or even validation loss) keeps improving indefinitely,
    should we always train as long as we can afford? Not necessarily.
    -   We might be able to tune more effectively by running a larger number of
        shorter experiments and reserving the longest "production length" runs
        for the models we hope to launch.
    -   As the training time for trials approaches our patience limit, tuning
        experiments become more relevant for our potential launch candidates,
        but we can complete fewer of them.
    -   There are probably many questions we can answer while only training for
        ~10% of the production length, but there is always a risk that our
        conclusions at this time limit will not apply to experiments at 20% of
        the production length, let alone 100%.
-   Tuning in multiple rounds with increasing, per-trial training step limits is
    a sensible approach.
    -   We can do as many rounds as we want, but usually 1-3 are the most
        practical.
    -   Essentially, try to obtain as much understanding of the problem as
        possible using trials with a very quick turnaround time, trading off
        tuning thoroughness with relevance to the final, longest runs.
    -   Once a given per-trial time limit has generated useful insights, we can
        increase the training time and continue tuning, double-checking our
        conclusions from the shorter runs as needed.
-   As a starting point, we recommend two rounds of tuning:
    -   Round 1: Shorter runs to find good model and optimizer hyperparameters.
    -   Round 2: Very few long runs on good hyperparameter points to get the
        final model.
-   The biggest question going from `Round i` &rarr; `Round i+1` is how to
    adjust learning rate decay schedules.
    -   One common pitfall when adjusting learning rate schedules between rounds
        is using all the extra training steps with too small of a learning rate.

#### Round 1

<details><summary><em>[Click to expand]</em></summary>

<br>

-   Unfortunately, there is no guarantee that good hyperparameters found in
    short, incomplete training are still good choices when training length is
    significantly increased. However, for some kinds of hyperparameters, they
    are often correlated enough for Round 1 to be useful.
-   What hyperparameter values found in shorter runs do we expect to transfer to
    longer training runs? For all of this, we need more research. But based on
    what we know so far, here are the authors’ suspicions in order of decreasing
    probability of transferring:
    -   Very likely to transfer
        -   Early training instability can be resolved in the first round of
            tuning using a smaller number of training steps. Perhaps these
            hyperparameters are the closest thing to a sure bet for transfer
            that we have.
            -   Warmup length
            -   Initialization
    -   Likely to transfer
        -   Model architecture - A dramatic win in the model architecture will
            usually transfer, but there are probably many counterexamples.
    -   Might transfer
        -   Optimization algorithm/optimizer hyperparameters - We think this
            would "loosely" transfer. It’s definitely weaker than the things
            above it.
        -   Data augmentation
        -   Regularization
            -   If it isn't possible to perfectly fit the training set, the
                model might be in a regime where regularization is unlikely to
                help very much.
    -   Unlikely to transfer
        -   Learning rate schedule: unlikely to transfer perfectly.
            -   [This paper](https://arxiv.org/abs/2203.15556) suggests that
                even decay schedule transfers, but we don't believe this is true
                in general. Example: Tuning sqrt decay on small # of training
                steps then extending to large # will result in the majority of
                training occurring at overly small steps.
                -   One can likely do "good enough" with most schedules in the
                    limit of extreme training budget, but noticeable performance
                    improvements can likely be seen if it is tuned.
            -   [Understanding Short-Horizon Bias in Stochastic
                Meta-Optimization](https://arxiv.org/abs/1803.02021) describes
                the dangers of trying to pick learning rates myopically.

</details>

#### Round 2

<details><summary><em>[Click to expand]</em></summary>

<br>

-   Run the best hyperparameter configuration from Round 1.
-   **(Speculation)** 🤖 Use the extra steps to extend the period of training at
    a high learning rate.
    -   E.g. if linear schedule then keep the length of the decay fixed from
        Round 1 and extend the period of constant lr in the beginning.
    -   For cosine decay, just keep the base lr from Round 1 and extend
        `max_train_steps` as in
        [Chinchilla paper](https://arxiv.org/abs/2203.15556).
-   More rounds might make sense for teams with very mature modeling and tuning
    pipelines and very long and expensive production training runs, but they
    will often be overkill.
    -   We've described how to transfer from Step 1 &rarr; Step 2. If we didn't care
        about analysis time and if making efficient use of compute was the
        overriding concern, then the ideal would be to exponentially increase
        the length of training runs (and thus the end-to-end time to complete a
        study) over many different rounds of tuning.
        -   At each round we systematically ensure our choices continue to hold
            up.
        -   New ideas go through a pipeline that progressively derisks them
            using increasingly long-running experiments from Step i to Step i+1.

</details>

## Additional guidance for the training pipeline

### Optimizing the input pipeline

***Summary:*** *The causes and interventions of input-bound pipelines are highly
task-dependent; use a profiler and look out for common issues.*

-   Use an appropriate profiler to diagnose input-bound pipelines. For example,
    [Perfetto](https://jax.readthedocs.io/en/latest/profiling.html) for JAX or
    [TensorFlow profiler](https://www.tensorflow.org/guide/profiler) for
    TensorFlow.
-   Ultimately, the specific causes and interventions will be highly
    task-dependent. Broader engineering considerations (e.g. minimizing disk
    footprint) may warrant worse input pipeline performance.
-   Common causes:
    -   Data are not colocated with the training process, causing I/O latency
        (this might happen when reading training data over a network).
    -   Expensive online data preprocessing (consider doing this once offline
        and saving).
    -   Unintentional synchronization barriers that interfere with data pipeline
        prefetching. For example, when synchronizing metrics between the device
        and host in CommonLoopUtils
        ([link](https://github.com/google/CommonLoopUtils/blob/fea2518ada8814a78e1492023fd9f00edb0b0568/clu/metrics.py#L291)).
-   Common tips:
    -   Instrument input pipeline to prefetch examples (e.g.
        [tf.data.Dataset.prefetch](https://www.tensorflow.org/guide/data_performance#prefetching))
    -   Remove unused features/metadata from each as early in the pipeline as
        possible.
    -   Increase the replication of the number of jobs generating examples for
        the input pipeline. For example, by using the
        [tf.data service](https://www.tensorflow.org/api_docs/python/tf/data/experimental/service).

### Evaluating model performance

***Summary:*** *Run evaluation at larger batch sizes than training. Run
evaluations at regular step intervals, not regular time intervals.*

#### Evaluation settings

<details><summary><em>[Click to expand]</em></summary>

<br>

-   There are several settings in which we can evaluate the performance of our
    models.
    -   **Online evaluation** - metrics are collected when the model is serving
        predictions in a production environment.
    -   **Offline evaluation** - metrics are collected when the model is run on
        offline train/validation/test sets that are representative of the
        production environment.
    -   **Periodic evaluations** - metrics are collected during model training
        that might either be a proxy for the offline evaluation, and/or on a
        subset of the data used in offline evaluation.
-   Online evaluation is the gold standard, but is often impractical during the
    model development phase.
-   Depending on the problem, offline evaluation can be fairly involved and
    computationally expensive.
-   Periodic evaluations are the most practical and economical choice, but may
    not fully represent the production environment.
    -   Our goal during periodic evaluation is to use an expedient proxy of the
        offline evaluation, without sacrificing the reliability of the signal we
        get during training.

</details>

#### Setting up periodic evaluations

<details><summary><em>[Click to expand]</em></summary>

<br>

-   We run periodic evaluations during training to monitor its progress in real
    time, to
    [facilitate retrospective model checkpoint selection](#saving-checkpoints-and-retrospectively-selecting-the-best-checkpoint),
    and so that we can
    [examine the training curves at the end of training](#examining-the-training-curves).
-   The simplest configuration is to perform both training and periodic
    evaluations within the same compute instance, periodically alternating
    between training and evaluation.
    -   In this case, the batch size used to perform evaluations should be *at
        least* as large as the batch size used for training because model
        activations don't need to be maintained during evaluation, lowering the
        computational requirements per example.
-   Periodic evaluations should be done at regular step intervals, not time
    intervals.
    -   Evaluating based on time intervals can make it harder to interpret the
        training curves, especially when training may suffer from preemptions of
        the training jobs, network latency issues, etc.
-   Periodicity in valid/test metrics (when using a shuffled
    train/validation/test split) can indicate implementation bugs such as test
    data having overlap with training data, or training data not being properly
    shuffled. Evaluating at regular step intervals can make these issues easier
    to catch.
-   Partial batches can occur when the evaluation sets are not divisible by the
    batch size. Ensure that the padded examples are correctly weighted to prevent
    the loss function from being biased by them. Often, these padded examples
    can be given a weight of zero.
-   Save sufficient information per evaluation to support offline analysis.
    Ideally, we would save predictions on a selection of individual examples
    since they can be invaluable for debugging.
    -   Generating artifacts like
        [SavedModels](https://www.tensorflow.org/guide/saved_model) make it easy
        to do ad-hoc model inspection after evaluation jobs finish.

</details>

#### Choosing a sample for periodic evaluation

<details><summary><em>[Click to expand]</em></summary>

<br>

-   The periodic evaluation job might not run fast enough to compute metrics on
    the full offline evaluation set in a reasonable amount of time. This often
    necessitates sampling data for periodic evaluation.
-   We consider the following factors when constructing a sampled dataset:
    -   <ins>Sample size</ins>
        -   Check that the performance computed on the sampled dataset used by
            the periodic job matches the performance on the whole offline
            evaluation set, i.e. there is no skew between the sampled set and
            the full dataset.
        -   The dataset used for periodic evaluation should be small enough that
            it’s easy to generate model predictions over its entirety, but large
            enough that improvements to the model can be accurately measured
            (i.e. not overwhelmed by label noise).
        -   It should be large enough to accommodate multiple such evaluations
            across trials in sequence, and still produce accurate estimates.
            That is, to avoid adaptively "fitting" to the validation set over
            time, in a way that doesn't generalize to a held-out test set.
            However, this consideration is rarely a practical concern.
    -   <ins>Imbalanced datasets</ins>
        -   For imbalanced datasets, performance on rare classes of examples
            will often be noisy.
        -   For datasets with a small number of examples in a class label, log
            the number of examples predicted correctly to get more insight into
            accuracy improvements (.05 sensitivity improvement sounds exciting,
            but was it just one more example correct?).

</details>

### Saving checkpoints and retrospectively selecting the best checkpoint

***Summary:*** *Run training for a fixed number of steps and retrospectively
choose the best checkpoint from the run.*

-   Most deep learning frameworks support
    [model checkpointing](https://flax.readthedocs.io/en/latest/api_reference/flax.training.html).
    That is, the current state of the model is periodically preserved on disk.
    This allows the training job to be resilient to compute instance
    interruptions.
-   The best checkpoint is often not the last checkpoint, particularly when the
    validation set performance does not continue to increase over time but
    rather fluctuates about a particular value.
-   Set up the pipeline to keep track of the N best checkpoints seen so far
    during training. At the end of training, model selection is then a matter of
    choosing the best checkpoint seen during training. We call this
    **retrospective optimal checkpoint selection**.
-   Supporting prospective early stopping is usually not necessary, since we’re
    pre-specifying a trial budget and are preserving the N best checkpoints seen
    so far.

### Setting up experiment tracking

***Summary:*** *When tracking different experiments, make sure to note a number
of essentials like the best performance of a checkpoint in the study, and a
short description of the study.*

-   We've found that keeping track of experiment results in a spreadsheet has
    been helpful for the sorts of modeling problems we've worked on. It often
    has the following columns:
    -   Study name
    -   A link to wherever the config for the study is stored.
    -   Notes or a short description of the study.
    -   Number of trials run
    -   Performance on the validation set of the best checkpoint in the study.
    -   Specific reproduction commands or notes on what unsubmitted changes were
        necessary to launch training.
-   Find a tracking system that captures at least the information listed above
    and is convenient for the people doing it. Untracked experiments might as
    well not exist.

### Batch normalization implementation details

***Summary:*** *Nowadays batch norm can often be replaced with LayerNorm, but in
cases where it cannot, there are tricky details when changing the batch size or
number of hosts.*

-   Batch norm normalizes activations using their mean and variance over the
    current batch, but in the multi-device setting these statistics are
    different on each device unless explicitly synchronized.
-   Anecdotal reports (mostly on ImageNet) say calculating these normalizing
    statistics using only ~64 examples actually works better in practice (see
    Ghost Batch Norm from [this paper](https://arxiv.org/abs/1705.08741)).
-   Decoupling the total batch size and the number of examples used to calculate
    batch norm statistics is particularly useful for batch size comparisons.
-   Ghost batch norm implementations do not always correctly handle the case
    where the per-device batch size > virtual batch size. In this case we'd
    actually need to subsample the batch on each device in order to get the
    proper number of batch norm statistic examples.
-   Exponential moving averages used in test mode batch norm are just a linear
    combination of training statistics, so these EMAs only need to be
    synchronized before saving them in checkpoints. However, some common
    implementations of batch norm do not synchronize these EMAs and only save
    the EMA from the first device.

### Considerations for multi-host pipelines

***Summary:*** *for logging, evals, RNGs, checkpointing, and data sharding,
multi-host training can make it very easy to introduce bugs!*

-   Ensure the pipeline is only logging and checkpointing on one host.
-   Make sure before evaluation or checkpointing is run, the batch norm
    statistics are synchronized across hosts.
-   It is critical to have RNG seeds that are the same across hosts (for model
    initialization), and seeds that are different across hosts (for data
    shuffling/preprocessing), so make sure to mark them appropriately.
-   Sharding data files across hosts is usually recommended for improved
    performance.

## FAQs

### What is the best learning rate decay schedule family?

<details><summary><em>[Click to expand]</em></summary>

<br>

-   It’s an open problem. It’s not clear how to construct a set of rigorous
    experiments to confidently answer what the "best" LR decay schedule is.
-   Although we don't know the best schedule family, we're confident that it’s
    important to have some (non-constant) schedule and that tuning it matters.
-   Different learning rates work best at different times during the
    optimization process. Having some sort of schedule makes it more likely for
    the model to hit a good learning rate.

</details>

### Which learning rate decay should I use as a default?

<details><summary><em>[Click to expand]</em></summary>
<br>

-   Our preference is either linear decay or cosine decay, and a bunch of other
    schedule families are probably good too.

</details>

### Why do some papers have complicated learning rate schedules?

<details><summary><em>[Click to expand]</em></summary>
<br>

-   It’s not uncommon to see papers with complicated piecewise learning rate
    (LR) decay schedules.
-   Readers often wonder how the authors arrived at such a complicated schedule.
-   Many complicated LR decay schedules are the result of tuning the schedule as
    a function of the validation set performance in an ad hoc way:
    1.  Start a single training run with some simple LR decay (or a constant
        learning rate).
    2.  Keep training running until the performance seems to stagnate. If this
        happens, pause training. Resume it with a perhaps steeper LR decay
        schedule (or smaller constant learning rate) from this point. Repeat
        this process until the conference/launch deadline.
-   Blithely copying the resulting *schedule* is generally not a good idea since
    the best particular schedule will be sensitive to a host of other
    hyperparameter choices.
    -   Better to copy the *algorithm* that produced the schedule, although this
        is rarely possible when arbitrary human judgment produced the schedule.
-   This type of validation-error-sensitive schedule is fine to use if it can be
    fully automated, but human-in-the-loop schedules that are a function of
    validation error are brittle and not easily reproducible, so we recommend
    avoiding them.
    -   Before publishing results that used such a schedule, please try to make
        it fully reproducible.

</details>

### How should Adam’s hyperparameters be tuned?

<details><summary><em>[Click to expand]</em></summary>
<br>

-   As discussed above, making general statements about search spaces and how
    many points one should sample from the search space is very difficult. Note
    that not all the hyperparameters in Adam are equally important. The
    following rules of thumb correspond to different "budgets" for the number of
    trials in a study.
    -   If < 10 trials in a study, only tune the (base) learning rate.
    -   If 10-25 trials, tune learning rate and $\beta_1$.
    -   If 25+ trials, tune the learning rate, $\beta_1$ and $\epsilon$.
    -   If one can run substantially more than 25 trials, additionally tune
        $\beta_2$.

</details>

### Why use quasi-random search instead of more sophisticated black box optimization algorithms during the exploration phase of tuning?

<details><summary><em>[Click to expand]</em></summary>

-   Quasi-random search (based on
    [low-discrepancy sequences](https://en.wikipedia.org/wiki/Low-discrepancy_sequence))
    is our preference over fancier black box optimization tools when used as
    part of an iterative tuning process intended to maximize insight into the
    tuning problem (what we refer to as the "exploration phase"). Bayesian
    optimization and similar tools are more appropriate for the exploitation
    phase.
-   Quasi-random search based on randomly shifted low-discrepancy sequences can
    be thought of as "jittered, shuffled grid search", since it uniformly, but
    randomly, explores a given search space and spreads out the search points
    more than random search.
-   The advantages of quasi-random search over more sophisticated black box
    optimization tools (e.g. Bayesian optimization, evolutionary algorithms)
    include:
    1.  Sampling the search space non-adaptively makes it possible to change the
        tuning objective in post hoc analysis without rerunning experiments.
        -   For example, we usually want to find the best trial in terms of
            validation error achieved at any point in training. But the
            non-adaptive nature of quasi-random search makes it possible to find
            the best trial based on final validation error, training error, or
            some alternative evaluation metric without rerunning any
            experiments.
    2.  Quasi-random search behaves in a consistent and statistically
        reproducible way.
        -   It should be possible to reproduce a study from six months ago even
            if the implementation of the search algorithm changes, as long as it
            maintains the same uniformity properties. If using sophisticated
            Bayesian optimization software, the implementation might change in
            an important way between versions, making it much harder to
            reproduce an old search. It isn’t always possible to roll back to an
            old implementation (e.g. if the optimization tool is run as a
            service).
    3.  Its uniform exploration of the search space makes it easier to reason
        about the results and what they might suggest about the search space.
        -   For example, if the best point in the traversal of quasi-random
            search is at the boundary of the search space, this is a good (but
            not foolproof) signal that the search space bounds should be
            changed. [This section](#identifying-bad-search-space-boundaries)
            goes into more depth. However, an adaptive black box optimization
            algorithm might have neglected the middle of the search space
            because of some unlucky early trials even if it happens to contain
            equally good points, since it is this exact sort of non-uniformity
            that a good optimization algorithm needs to employ to speed up the
            search.
    4.  Running different numbers of trials in parallel versus sequentially will
        not produce statistically different results when using quasi-random
        search (or other non-adaptive search algorithms), unlike with adaptive
        algorithms.
    5.  More sophisticated search algorithms may not always handle infeasible
        points correctly, especially if they aren't designed with neural network
        hyperparameter tuning in mind.
    6.  Quasi-random search is simple and works especially well when many tuning
        trials will be running in parallel.
        -   Anecdotally[^3], it is very hard for an adaptive algorithm to beat a
            quasi-random search that has 2X its budget, especially when many
            trials need to be run in parallel (and thus there are very few
            chances to make use of previous trial results when launching new
            trials).
        -   Without expertise in Bayesian optimization and other advanced black
            box optimization methods, we might not achieve the benefits they
            are, in principle, capable of providing. It is hard to benchmark
            advanced black box optimization algorithms in realistic deep
            learning tuning conditions. They are a very active area of current
            research, and the more sophisticated algorithms come with their own
            pitfalls for inexperienced users. Experts in these methods are able
            to get good results, but in high-parallelism conditions the search
            space and budget tend to matter a lot more.
-   That said, if our computational resources only allow a small number of
    trials to run in parallel and we can afford to run many trials in sequence,
    Bayesian optimization becomes much more attractive despite making our tuning
    results harder to interpret.

[^3]: Ben Recht and Kevin Jamieson
    [pointed out](http://www.argmin.net/2016/06/20/hypertuning/) how strong
    2X-budget random search is as a baseline (the
    [Hyperband paper](https://jmlr.org/papers/volume18/16-558/16-558.pdf)
    makes similar arguments), but it is certainly possible to find search
    spaces and problems where state-of-the-art Bayesian optimization
    techniques crush random search that has 2X the budget. However, in our
    experience beating 2X-budget random search gets much harder in the
    high-parallelism regime since Bayesian optimization has no opportunity to
    observe the results of previous trials.

</details>

### Where can I find an implementation of quasi-random search?

<details><summary><em>[Click to expand]</em></summary>
<br>

-   [Open-Source Vizier](https://github.com/google/vizier) has an [implementation
    of quasi-ranom search](https://github.com/google/vizier/blob/main/vizier/_src/algorithms/designers/quasi_random.py). Set `algorithm="QUASI_RANDOM_SEARCH"` in [this usage example](https://oss-vizier.readthedocs.io/en/latest/guides/user/running_vizier.html).
-   An alternative implementation exists
    [here](https://github.com/mlcommons/algorithmic-efficiency/blob/main/algorithmic_efficiency/halton.py).
-   Both implementations above generate a Halton sequence for a given search space (intended to
    implement a shifted, scrambled Halton sequence as recommended in
    https://arxiv.org/abs/1706.03200).
-   If a quasi-random search algorithm based on a low-discrepancy sequence is
    not available, it is possible to substitute pseudo random uniform search
    instead, although this is likely to be slightly less efficient.
    -   In 1-2 dimensions, grid search is also acceptable, although not in
        higher dimensions (see
        [Bergstra & Bengio, 2012](https://www.jmlr.org/papers/v13/bergstra12a.html)).

</details>

### How many trials are needed to get good results with quasi-random search?

<details><summary><em>[Click to expand]</em></summary>
<br>

<p align="center">
<img src="assets/validation_error_vs_num_trials.png" width="55%" alt="A box plot showing the importance of sampling enough">
</p>

<p align="center"><b>Figure 3:</b> A ResNet-50 was tuned on ImageNet with 100
trials. Via bootstrapping, different amounts of tuning budget were simulated.
Box plots of the best performances for each trial budget are plotted above.

-   There is no way to answer this question in general, but we can look at
    specific examples.
-   As the Figure 3 shows, the number of trials in a study can have a
    substantial impact on the results.
    -   Notice how large the interquartile ranges are when 6 trials were
        sampled, versus when 20 trials were sampled.
    -   Even with 20 trials, it is likely that the difference between especially
        lucky and unlucky studies will be larger than the typical variation
        between re-trains of this model on different random seeds, with fixed
        hyperparameters, which for this workload might be around +/- 0.1% on a
        validation error rate of \~23%.

</details>

### How can optimization failures be debugged and mitigated?

<details><summary><em>[Click to expand]</em></summary>
<br>


***Summary:*** *If the model is experiencing optimization difficulties, it’s
important to fix them before trying other things. Diagnosing and correcting
training failures is an active area of research.*

<p align="center">
<img src="assets/stride_instability.png" width="80%" alt="Changing the strides in a single residual block in a WideResnet results in training instability.">
</p>


<p align="center"><b>Figure 4:</b> Changing the strides in a single residual block (2x2 -> 1x1) in a WideResnet results in training instability. This does not degrade performance at low learning rates, but high learning rates no longer train well due to the instability. Applying 1000 steps of learning rate warmup resolves this particular instance of instability, allowing stable training at max learning rate of .1.</p>

#### Identifying unstable workloads

-   Any workload will become unstable if the learning rate is too large.
    Instability is only an issue when it forces us to use a learning rate that’s
    too small.
-   There are at least two types of training instability worth distinguishing:
    1.  Instability at initialization/early in training.
    2.  Sudden instability in the middle of training.
-   We can take a systematic approach to identifying stability issues in our
    workload.
    1.  Do a learning rate sweep and find the best learning rate lr*.
    2.  Plot training loss curves for learning rates just above lr*.
    3.  If the learning rates > lr* show loss instability (loss goes up not down
        during periods of training), then it is likely that fixing the
        instability will result in better training.
-   Log the L2 norm of the full loss gradient during training, outlier values
    can result in spurious instability in the middle of training. This can
    inform how to pick gradient/update clipping.

**NOTE:** Some models show very early instability followed by a recovery that
results in slow but stable training. **Common evaluation schedules can miss
these issues by not evaluating frequently enough!**

To check for this, we can train for an abbreviated run of just \~500 steps using
`lr = 2 * current best`, but evaluate every step.

<p align="center">
<img src="assets/more_frequent_evals.png" width="80%" alt="Illustration of the value of more frequent evaluations at the start of
training.">
</p>

<p align="center"><b>Figure 5:</b> Illustration of the value of more frequent evaluations at the start of training. Useful if there’s a suspicion that the model suffers from early training instability.</p>

#### Potential fixes for common instability patterns

-   Apply learning rate warmup
    -   Best for early training instability.
-   Apply gradient clipping
    -   Good for both early and mid training instability, may fix some bad inits
        that warmup cannot.
-   Try a new optimizer
    -   Sometimes Adam can handle instabilities that Momentum can’t. This is an
        active area of research.
-   We can ensure that we’re using best practices/initializations for our model
    architecture (examples below).
    -   Add residual connections and normalization if the model doesn't contain
        it already.
-   Normalization should be the last operation before the residual. E.g. x +
    Norm(f(x)).
-   Norm(x + f(x)) known to cause issues.
-   Try initializing residual branches to 0 (e.g.
    [ReZero init](https://arxiv.org/abs/2003.04887)).
-   Lower the learning rate
    -   This is a last resort.

#### Learning rate warmup

<p align="center">
<img src="assets/instability_during_warmup.png" width="80%" alt="An example of instability during a warmup period (note the horizontal axis log
scale).">
</p>

<p align="center"><b>Figure 6:</b> An example of instability during a warmup period (note the horizontal axis log scale). 40k steps of warmup was needed for successful training in this case.</p>

##### When to apply learning rate warmup

<p align="center">
<img src="assets/axis_model_with_instability.png" width="49%" alt="Axis plot for model with instability">
</p>

<p align="center"><b>Figure 7a:</b> An example of a hyperparameter axis plot for a model exhibiting training instability. The best learning rate is at the edge of what is feasible. An "infeasible" trial is defined as one that either produces NaNs or uncharacteristically high values of the loss.</p>

<p align="center">
<img src="assets/loss_model_with_instability.png" width="49%" alt="Loss curve for model with instability">
</p>

<p align="center"><b>Figure 7b:</b> The training loss of a model trained with a learning rate where we see instability.</p>

-   Figure 7a shows a hyperparameter axis plot that indicates a model
    experiencing optimization instabilities, because the best learning rate is
    right at the edge of instability.
-   Figure 7b shows how this can be double-checked by examining the training
    loss of a model trained with a learning rate either 5x or 10x larger than
    this peak. If that plot shows a sudden rise in the loss after a steady
    decline (e.g. at step \~10k in the figure above), then the model likely
    suffers from optimization instability.

##### How to apply learning rate warmup

<p align="center">
<img src="assets/beneficial_effect_warmup.png" width="80%" alt="Beneficial effect of warmup on training instabilities">
</p>

<p align="center"><b>Figure 8:</b> Beneficial effect of learning rate warmup on addressing training instabilities.</p>

-   Using the section immediately above, we assume that the practitioner has
    already identified the learning rate at which the model becomes unstable.
    This is the `unstable_base_learning_rate`.
-   Warmup involves prepending a learning rate schedule that ramps up the
    learning rate from 0 to some stable `base_learning_rate`, that is at least
    one order of magnitude larger than `unstable_base_learning_rate`. The
    default would be to try a `base_learning_rate` that’s 10x
    `unstable_base_learning_rate`. Although note that it’d be possible to run
    this entire procedure again for something like 100x
    `unstable_base_learning_rate`. The specific schedule is:
    -   Ramp up from 0 to `base_learning_rate` over `warmup_steps`.
    -   Train at a constant rate for `post_warmup_steps`.
-   Our goal is to find the shortest number of `warmup_steps` that allows us to
    access peak learning rates that are much higher than
    `unstable_base_learning_rate`.
-   So for each `base_learning_rate`, we need to tune `warmup_steps` and
    `post_warmup_steps`. It’s usually fine to set `post_warmup_steps` to be
    `2*warmup_steps`.
-   Warmup can be tuned independently of an existing decay schedule.
    `warmup_steps` should be swept at a few different orders of magnitude. For
    example, an example study could try [10, 10<sup>3</sup>, 10<sup>4</sup>,
    10<sup>5</sup>]. The largest feasible point shouldn't be more than 10% of
    `max_train_steps`.
-   Once a `warmup_steps` that doesn't blow up training at `base_learning_rate`
    has been established, it should be applied to the baseline model.
    Essentially, we prepend this schedule onto the existing schedule, and use
    the optimal checkpoint selection discussed above to compare this experiment
    to the baseline. For example, if we originally had 10,000 `max_train_steps`
    and did `warmup_steps` for 1000 steps, the new training procedure should run
    for 11,000 steps total.
-   If long `warmup_steps` are required for stable training (>5% of
    `max_train_steps`), `max_train_steps` may need to be increased to account
    for this.
-   There isn't really a "typical" value across the full range of workloads.
    Some models only need 100 steps, while others (particularly transformers)
    may need 40k+.

#### Gradient clipping

<p align="center">
<img src="assets/gradient_clipping.png" width="80%" alt="Gradient clipping on early training instabilities">
</p>

<p align="center"><b>Figure 9:</b> Illustration of gradient clipping correcting early training instability.</p>

-   Gradient clipping is most useful when large or outlier gradient issues
    occur.
-   Clipping can fix either early training instability (large gradient norm
    early), or mid training instabilities (sudden gradient spikes mid training).
-   Sometimes longer warmup periods can correct instabilities that clipping does
    not: see [this section above](#How-to-apply-learning-rate-warmup).
    -   🤖 What about clipping during warmup?
-   The ideal clip thresholds are just above the "typical" gradient norm.
-   Here’s an example of how gradient clipping could be done:
    -   If the norm of the gradient $\left | g \right |$ is greater than the
        gradient clipping threshold $\lambda$, then do ${g}'= \lambda \times \frac{g}{\left | g \right |}$ where ${g}'$ is the new gradient.
-   Log the unclipped gradient norm during training. By default, generate:
    -   A plot of gradient norm vs step
    -   A histogram of gradient norms aggregated over all steps
-   Choose a gradient clipping threshold based on the 90th percentile of
    gradient norms.
    -   The threshold will be workload dependent, but 90% is a good starting
        point. If it doesn't work, this threshold can be tuned.
    -   🤖 What about some sort of adaptive strategy?
-   If we try gradient clipping and the instability issues remain, we can try it
    harder (i.e. make the threshold smaller).
-   Extremely aggressive gradient clipping is in essence a strange way of
    reducing the learning rate. If we find ourselves using extremely aggressive
    clipping, we probably should just cut the learning rate instead.
-   We would usually consider having >50% of the updates getting clipped somehow
    as "extremely aggressive".
-   If we need to do extremely aggressive gradient clipping to deal with our
    instability issues, then we might as well reduce the learning rate.

</details>

### Why do you call the learning rate and other optimization parameters hyperparameters? They are not parameters of any prior distribution.

<details><summary><em>[Click to expand]</em></summary>
<br>

-   It is true that the term "hyperparameter" has a precise
    [meaning](https://en.wikipedia.org/wiki/Hyperparameter) in Bayesian machine
    learning and referring to the learning rate and most of the other parameters
    we tune in deep learning as "hyperparameters" is an abuse of terminology.
-   We would prefer to use the term "metaparameter" for learning rates,
    architectural parameters, and all the other things we tune in deep learning,
    since it avoids the potential for confusion that comes from misusing the
    word "hyperparameter" (confusion that is especially likely when discussing
    Bayesian optimization where the probabilistic response surface models have
    their own true hyperparameters).
-   Unfortunately, although potentially confusing, the term hyperparameter has become
    extremely common in the deep learning community.
-   Therefore, for a document, such as this one, intended for a wide audience
    that includes many people who are unlikely to be aware of this technicality,
    we made the choice to contribute to one source of confusion in the
    field in hopes of avoiding another.
-   That said, we might make a different choice when publishing a research
    paper, and we would encourage others to use "metaparameter" instead in most
    contexts.

</details>

### Why shouldn't the batch size be tuned to directly improve validation set performance?

<details><summary><em>[Click to expand]</em></summary>
<br>

-   Changing the batch size *without changing any other details of the training pipeline* will often affect the validation set performance.
-   However, the difference in validation set performance between two batch sizes typically goes away if the training pipeline is optimized independently for each batch size.
-   The hyperparameters that interact most strongly with the batch size, and therefore are most important to tune separately for each batch size, are the optimizer hyperparameters (e.g. learning rate, momentum) and the regularization hyperparameters.
    - Smaller batch sizes introduce more noise into the training algorithm due to sample variance, and this noise can have a regularizing effect. Thus, larger batch sizes can be more prone to overfitting and may require stronger regularization and/or additional regularization techniques.
- In addition, [the number of training steps may need to be adjusted](#choosing-the-batch-size-to-minimize-training-time) when changing the batch size.
-   Once all these effects are taken into account, there is currently no convincing evidence that the batch size affects the maximum achievable validation performance (see [Shallue et al. 2018](https://arxiv.org/abs/1811.03600)).

</details>

### What are the update rules for all the popular optimization algorithms?

<details><summary><em>[Click to expand]</em></summary>

<br>

#### Stochastic gradient descent (SGD)

$$\theta_{t+1} = \theta_{t} - \eta_t \nabla \mathcal{l}(\theta_t)$$

#### Momentum

$$v_0 = 0$$

$$v_{t+1} = \gamma v_{t} + \nabla \mathcal{l}(\theta_t)$$

$$\theta_{t+1} = \theta_{t} - \eta_t v_{t+1}$$

#### Nesterov

$$v_0 = 0$$

$$v_{t+1} = \gamma v_{t} + \nabla \mathcal{l}(\theta_t)$$

$$\theta_{t+1} = \theta_{t} - \eta_t( \gamma v_{t+1} + \nabla \mathcal{l}(\theta_{t})$$

#### RMSProp

$$v_0 = 1 \text{,} m_0 = 0$$

$$v_{t+1} = \rho v_{t} + (1 - \rho) \nabla \mathcal{l}(\theta_t)^2$$

$$m_{t+1} = \gamma m_{t} + \frac{\eta_t}{\sqrt{v_{t+1} + \epsilon}}\nabla \mathcal{l}(\theta_t)$$

$$\theta_{t+1} = \theta_{t} - m_{t+1}$$

#### ADAM

$$m_0 = 0 \text{,} v_0 = 0$$

$$m_{t+1} = \beta_1 m_{t} + (1 - \beta_1) \nabla \mathcal{l} (\theta_t)$$

$$v_{t+1} = \beta_2 v_{t} + (1 - \beta_2) \nabla \mathcal{l}(\theta_t)^2$$

$$b_{t+1} = \frac{\sqrt{1 - \beta_2^{t+1}}}{1 - \beta_1^{t+1}}$$

$$\theta_{t+1} = \theta_{t} - \alpha_t \frac{m_{t+1}}{\sqrt{v_{t+1}} + \epsilon} b_{t+1}$$

#### NADAM

$$m_0 = 0 \text{,} v_0 = 0$$

$$m_{t+1} = \beta_1 m_{t} + (1 - \beta_1) \nabla \mathcal{l} (\theta_t)$$

$$v_{t+1} = \beta_2 v_{t} + (1 - \beta_2) \nabla \mathcal{l} (\theta_t)^2$$

$$b_{t+1} = \frac{\sqrt{1 - \beta_2^{t+1}}}{1 - \beta_1^{t+1}}$$

$$\theta_{t+1} = \theta_{t} - \alpha_t \frac{\beta_1 m_{t+1} + (1 - \beta_1) \nabla \mathcal{l} (\theta_t)}{\sqrt{v_{t+1}} + \epsilon} b_{t+1}$$

</details>

## Acknowledgments

-   We owe a debt of gratitude to Max Bileschi, Roy Frostig, Zelda Mariet, Stan
    Bileschi, Mohammad Norouzi, Chris DuBois and Charles Sutton for reading the
    manuscript and providing valuable feedback.
-   We reused some experimental data for several plots that were originally
    produced by Naman Agarwal for other joint research.
-   We would like to thank Will Chen for invaluable advice on the presentation of the document.
-   We would also like to thank Rohan Anil for useful discussions.

## Citing

```
@misc{tuningplaybookgithub,
  author = {Varun Godbole and George E. Dahl and Justin Gilmer and Christopher J. Shallue and Zachary Nado},
  title = {Deep Learning Tuning Playbook},
  url = {http://github.com/google-research/tuning_playbook},
  year = {2023},
  note = {Version 1.0}
}
```

## Contributing

-   This is not an officially supported Google product.

-   We'd love to hear your feedback!

    -   If you like the playbook, please [leave a star](https://docs.github.com/en/get-started/exploring-projects-on-github/saving-repositories-with-stars#starring-a-repository)! Or email
        deep-learning-tuning-playbook \[at\] googlegroups.com. Testimonials help
        us justify creating more resources like this.
    -   If anything seems incorrect, please file an issue to start a discussion.
        For questions or other messages where an issue isn't appropriate, please
        open a new discussion topic on GitHub.

-   As discussed in the preamble, this is a living document. We anticipate
    making periodic improvements, both small and large. If you’d like to be
    notified, please watch our repository (see [instructions](https://docs.github.com/en/account-and-profile/managing-subscriptions-and-notifications-on-github/setting-up-notifications/configuring-notifications#configuring-your-watch-settings-for-an-individual-repository)).

-   Please don't file a pull request without first coordinating with the authors
    via the issue tracking system.

### Contributor License Agreement

Contributions to this project must be accompanied by a Contributor License
Agreement (CLA). You (or your employer) retain the copyright to your
contribution; this simply gives us permission to use and redistribute your
contributions as part of the project. Head over to
<https://cla.developers.google.com/> to see your current agreements on file or
to sign a new one.

You generally only need to submit a CLA once, so if you've already submitted one
(even if it was for a different project), you probably don't need to do it
again.

### Code Reviews

All submissions, including submissions by project members, require review. We
use GitHub pull requests for this purpose. Consult
[GitHub Help](https://help.github.com/articles/about-pull-requests/) for more
information on using pull requests.

### Community Guidelines

This project follows
[Google's Open Source Community Guidelines](https://opensource.google/conduct/).
