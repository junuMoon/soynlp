# 8. 최종 재구성 제안서

이 문서는 현재 `soynlp` 구현을 유지보수 관점에서 다시 읽고, WordRank, KR-WordRank, 그리고 이 레포의 핵심 아이디어를 하나의 공통 설계 위에서 다시 세운다면 어떤 구조가 가장 자연스러운지를 제안하는 문서다.

처음에는 이 주제로 개별 제안서와 통합 결정안을 따로 만들었지만, 설계 문서가 둘로 갈라져 있으면 오히려 읽는 흐름이 깨진다. 그래서 지금 문서는 내 초안과 두 개의 `gpt-5.4 xhigh` 설계 초안에서 공통으로 수렴한 내용을 하나로 합친 `최종판`이다. 이제부터는 이 문서를 설계 제안의 단일 기준 문서로 본다.

목표는 "조금 더 깔끔하게 리팩터링" 수준이 아니다. 오히려 아래 질문에 답하는 쪽에 가깝다.

- 이 레포의 진짜 공통 엔진은 무엇인가?
- WordRank와 KR-WordRank와 noun/predicator/pos 추출기를 하나의 계보로 묶는 추상화는 무엇인가?
- 현재 코드의 장점은 살리되, 왜 구조가 점점 복잡해졌는지를 어떻게 해소할 수 있는가?
- 연구용 알고리즘 실험과 실사용 API를 동시에 품으려면 어떤 계층화가 맞는가?

세 초안이 공통으로 동의한 결론은 아래였다.

- WordRank, KR-WordRank, soynlp noun/predicator는 서로 다른 제품이 아니라 같은 family다.
- `LRGraph`는 유틸이 아니라 그래프 계층의 중심 엔진이어야 한다.
- 형태 복원은 별도 morphology layer로 분리되어야 한다.
- 그래프를 직접 지우는 mutable shared state보다, 원본 그래프와 별도 coverage/state 계층이 더 낫다.
- 기능별 패키지보다 `관측 -> 그래프 -> 랭킹/분류 -> 복원 -> 디코더` 구조가 더 자연스럽다.

## 8.1 먼저 한 문장으로 요약

내 제안은 이 레포를 "한국어 비지도 NLP 도구 모음"이 아니라, `boundary-aware lexical graph platform`으로 다시 정의하는 것이다.

즉 중심은 더 이상 `noun`, `predicator`, `tokenizer` 같은 개별 기능 패키지가 아니다. 중심은 아래다.

```text
문장 관측
-> 경계/부분문자열 관측 저장소
-> 타입 있는 그래프/랭킹/분류 엔진
-> 형태 복원 엔진
-> 분석기와 파이프라인
```

이렇게 바꾸면 WordRank, KR-WordRank, 명사 추출, 용언 추출, POS 추출이 서로 다른 발명품이 아니라 같은 플랫폼 위의 다른 analyzer로 보이기 시작한다.

## 8.2 왜 지금 구조가 아쉬운가

현재 레포의 가장 큰 강점은 알고리즘 아이디어가 살아 있다는 점이다. 반면 구조적으로는 다음 문제가 있다.

### 1. 공통 엔진이 패키지 안쪽에 숨어 있다

실제로 많은 알고리즘이 공유하는 것은 아래다.

- eojeol 단위 관측
- substring/prefix/suffix counting
- L/R boundary graph
- 반복적인 후보 확정과 그래프 소비
- 표면형과 기본형의 왕복 복원

그런데 현재 구조에서는 이 공통 엔진이 `utils`, `noun`, `predicator`, `word`, `pos` 곳곳에 퍼져 있다.

### 2. "알고리즘"과 "파이프라인"이 분리되지 않았다

예를 들어 `LRNounExtractor_v2`는 아래 역할을 동시에 맡고 있다.

- 데이터 적재
- predictor asset 로딩
- 후보 생성
- 점수 계산
- 그래프 소비
- compound 후처리
- 로그 저장

이렇게 되면 하나의 아이디어를 재사용하기 어렵고, 실험 단위가 너무 크다.

### 3. 상태 변화가 암묵적이다

현재 구현의 중요한 특징 중 하나는 강한 후보를 찾으면 `LRGraph`에서 관련 eojeol을 제거한다는 점이다. 이건 알고리즘적으로 매우 좋은 선택인데, 코드 구조상으로는 "언제 무엇을 왜 소비했는가"가 잘 드러나지 않는다.

즉 현재는 결과는 나오지만, 설명 가능성과 재현성이 구조 차원에서 강하지 않다.

### 4. WordRank/KR-WordRank/soynlp가 한 가문이라는 사실이 코드에서 보이지 않는다

개념적으로는 다음처럼 볼 수 있다.

- WordRank: 무타입 경계 그래프 랭킹
- KR-WordRank: L/R 타입이 붙은 경계 그래프 랭킹
- `LRNounExtractor_v2`: seed feature를 가진 typed boundary classifier
- `PredicatorExtractor`: boundary evidence + morphology resolver
- `NewsPOSExtractor`: analyzer orchestration

그런데 현재 코드는 이 관계를 파일 구조로 보여 주지 못한다.

## 8.3 재구성의 핵심 원칙

새 구조는 아래 원칙을 따라야 한다.

### 1. 관측과 해석을 분리한다

문장으로부터 어떤 통계를 관측하는 단계와, 그 통계를 가지고 무엇을 명사/용언/키워드로 해석하는 단계는 분리해야 한다.

### 2. 그래프를 1급 추상화로 올린다

`LRGraph`는 유틸이 아니라 핵심 엔진이다. 새 구조에서는 boundary graph를 플랫폼 중심에 둬야 한다.

### 3. 형태 복원을 별도 엔진으로 독립시킨다

`lemma_candidate`, `conjugate`, stem/eomi validation은 noun extractor의 하위 세부사항이 아니라 별도 morphology layer여야 한다.

### 4. 상태 소비는 명시적으로 남긴다

그래프에서 edge나 eojeol을 소비했다면, 그것을 결과와 함께 기록해야 한다.

### 5. 알고리즘은 조립형이어야 한다

새 noun analyzer를 만들 때 클래스 하나를 복사해 고치는 대신,

- candidate generator
- scorer
- pruner
- resolver
- explainer

를 조합하는 방식으로 가야 한다.

### 6. 연구용 실험과 호환 API를 분리한다

현재 레포를 쓰는 사용자 입장에서는 익숙한 API가 필요하다. 반면 내부에서는 완전히 다른 구조가 더 낫다. 따라서 새 구조에는 `compat` 계층이 필요하다.

## 8.4 새 레포의 중심 추상화

내가 새로 세울 때 가장 먼저 만드는 객체는 아래 다섯 가지다.

### `CorpusView`

문장 스트림과 문서 경계를 표현하는 가장 얇은 읽기 계층이다.

책임:

- raw text iteration
- 문서/문장 단위 구분
- 메타데이터 보존
- lazy loading

즉 현재 `DoublespaceLineCorpus`보다 조금 더 일반적인 입구다.

### `ObservationStore`

이 설계의 핵심 저장소다. 현재 `EojeolCounter`, `WordExtractor`의 내부 카운터, `LRGraph` 생성 전후 통계를 통합하는 역할을 맡는다.

책임:

- eojeol frequency
- substring frequency
- left/right boundary counts
- neighbor character statistics
- optional co-occurrence matrices
- normalization provenance

이 객체는 "한 번 만들고 여러 analyzer가 공유"해야 한다.

### `BoundaryGraph`

현재 `LRGraph`를 일반화한 그래프 계층이다.

기본 형태:

- node: 문자열 조각
- edge: `left_part -> right_part`
- metadata: count, source span, typed tag, normalization state

하위 변형:

- `UntypedBoundaryGraph`: WordRank용
- `TypedBoundaryGraph`: KR-WordRank, noun/predicator용
- `ConsumableBoundaryGraph`: 단계적 edge 소비가 필요한 analyzer용

즉 새 구조에서는 `LRGraph`가 더 이상 utils 안의 helper가 아니라 graph layer의 핵심 타입이 된다.

### `MorphResolver`

형태 복원을 별도로 담당한다.

책임:

- `conjugate`
- `lemma_candidate`
- stem surface generation
- eomi surface validation
- adjective/verb split support

즉 `predicator`와 `lemmatizer`를 다시 생각할 때, 핵심은 "용언 추출기"가 아니라 "형태 복원 서비스"다.

### `AnalysisState`

이 객체는 각 analyzer가 현재까지 무엇을 설명했고, 무엇을 소비했고, 무엇이 남았는지를 명시적으로 기록한다.

예시 필드:

- `active_graph`
- `consumed_edges`
- `consumed_eojeols`
- `accepted_candidates`
- `rejected_candidates`
- `decision_log`

이게 있으면 현재처럼 `reset_lrgraph()`와 `remove_eojeol()`의 의미를 머리로 추적할 필요가 줄어든다.

## 8.5 WordRank, KR-WordRank, soynlp를 한 계보로 다시 정의하기

새 구조에서는 세 계열 알고리즘을 아래처럼 재정의한다.

### WordRank

```text
입력: UntypedBoundaryGraph
연산: graph ranking + cohesion/internal binding
출력: surface token candidates
```

즉 타입 없는 경계 그래프 랭커다.

### KR-WordRank

```text
입력: TypedBoundaryGraph(L/R)
연산: typed graph ranking + R-based pruning
출력: keyword/lexical candidates
```

즉 L/R 타입이 붙은 WordRank다.

### soynlp noun extractor

```text
입력: TypedBoundaryGraph + seeded R feature lexicon
연산: positive/common/negative feature classification + graph consumption + compound recovery
출력: noun candidates with evidence
```

즉 KR-WordRank의 "R을 후처리에 쓰는 아이디어"를 훨씬 더 강한 supervision-like feature 체계로 확장한 분석기라고 볼 수 있다.

### soynlp predicator extractor

```text
입력: residual boundary graph + MorphResolver + noun decisions
연산: eomi extraction + stem extraction + lemma validation
출력: predicator candidates
```

즉 형태 복원 계층과 경계 그래프 계층이 만나는 곳이다.

### soynlp POS extractor

```text
입력: noun/predicator outputs + residual eojeols
연산: staged matching and reconciliation
출력: typed lexicon
```

즉 상위 orchestration layer다.

이 정의가 중요한 이유는, 앞으로 새 알고리즘을 추가할 때도 "이건 어떤 graph analyzer인가?"로 설명할 수 있기 때문이다.

## 8.6 새 패키지 구조 제안

아래는 내가 새로 짠다면 제안하는 패키지 구조다.

```text
soynlp/
  core/
    corpus.py
    normalize.py
    observation.py
    evidence.py
    state.py

  graph/
    boundary.py
    typed_boundary.py
    consumable.py
    builders.py

  rank/
    wordrank.py
    kr_wordrank.py
    cohesion.py
    entropy.py
    pmi.py

  morph/
    conjugation.py
    lemmatization.py
    resolver.py
    surfaces.py
    predicates.py

  analyzers/
    noun.py
    predicator.py
    pos.py
    tokenizer.py
    keywords.py

  pipelines/
    noun_pipeline.py
    predicator_pipeline.py
    pos_pipeline.py
    keyword_pipeline.py

  assets/
    loaders.py
    noun_features.py
    dictionaries.py

  compat/
    noun_v2.py
    predicator.py
    pos.py
    tokenizer.py

  diagnostics/
    explain.py
    report.py
    visualize.py
```

핵심은 세 층이다.

- `core/graph/morph`: 공통 엔진
- `analyzers/pipelines`: 알고리즘 조립
- `compat`: 기존 공개 API 유지

## 8.7 실제로는 어떤 인터페이스가 필요할까

새 구조에서 중요한 것은 클래스 이름보다 인터페이스다.

### 공통 analyzer 인터페이스

```python
class Analyzer:
    def fit(self, observations, state=None): ...
    def propose(self, state): ...
    def score(self, proposals, state): ...
    def accept(self, scored, state): ...
    def explain(self, candidate_id): ...
```

이 인터페이스가 있으면 noun, predicator, KR-WordRank analyzer가 동일한 생명주기를 공유하게 된다.

### 결과 객체

현재 결과는 dict, namedtuple, 내부 상태가 뒤섞여 있다. 새 구조에서는 아래처럼 통일하는 편이 좋다.

```python
Candidate(
    text="아이오아이",
    kind="noun",
    score=0.87,
    frequency=152,
    lemma=None,
    evidence=[
        Evidence(type="pos_feature", value="는", count=31),
        Evidence(type="eojeol_end", value="", count=90),
    ],
)
```

즉 결과가 단순 숫자만이 아니라 `왜 그렇게 판단했는가`를 함께 가져야 한다.

### 상태 소비 로그

```python
Decision(
    analyzer="noun",
    candidate="아이오아이",
    action="consume_edges",
    edges=[("아이오아이", "는"), ("아이오아이", "")],
    reason="score >= threshold and positive_feature_support",
)
```

이런 로그가 있으면 디버깅, 설명, 실험 비교가 쉬워진다.

## 8.8 noun/predicator/pos를 새 구조로 다시 쓰면

### 1. noun analyzer

현재의 `LRNounExtractor_v2`는 아래 네 조각으로 분해할 수 있다.

- `NounCandidateGenerator`
- `BoundaryFeatureScorer`
- `GraphConsumptionPolicy`
- `CompoundRecoveryPass`

이렇게 쪼개면 "compound를 끄고 base noun만 보고 싶다" 같은 실험이 쉬워진다.

### 2. predicator analyzer

현재 `PredicatorExtractor`는 사실 세 analyzer의 묶음이다.

- `ResidualEojeolSelector`
- `EomiAnalyzer`
- `StemAnalyzer`
- `LemmaValidator`

새 구조에서는 이들을 조립형 pipeline으로 바꿔야 한다.

### 3. POS pipeline

현재 `NewsPOSExtractor`는 좋은 아이디어를 가졌지만 로직이 긴 한 파일에 몰려 있다. 새 구조에서는 단계가 명시적으로 나뉘어야 한다.

```text
WordMatchPass
-> NounPlusRPass
-> PredicatorCompoundPass
-> LemmaRecoveryPass
-> OneSyllableNounPass
-> IrregularCleanupPass
-> CompoundNounPass
```

이렇게 되면 각 pass를 독립적으로 테스트하고 켜고 끌 수 있다.

## 8.9 tokenizer를 어떻게 다시 위치시킬까

현재 tokenizer는 다소 독립 패키지처럼 보이지만, 사실은 이 레포의 공통 점수 체계를 소비하는 downstream layer다.

새 구조에서는 tokenizer를 두 갈래로 나눈다.

### `score_based_tokenizer`

- `WordRank`나 noun score를 받아 greedy 분해
- 현재 `LTokenizer`, `MaxScoreTokenizer` 계열을 포함

### `graph_aware_tokenizer`

- typed boundary graph와 preference를 함께 사용
- 현재 `MaxLRScoreTokenizer`의 실험선을 여기로 이동

즉 tokenizer는 더 이상 독립 철학이 아니라, graph/rank 계층을 소비하는 application layer가 된다.

## 8.10 무엇을 남기고 무엇을 버릴까

재구성이라고 해서 모든 것을 새로 쓰는 건 아니다. 오히려 아래는 반드시 남겨야 한다.

### 남길 것

- `EojeolCounter`의 단순하고 빠른 counting 감각
- `LRGraph`의 직관적인 L/R 모델
- `lemma_candidate()`의 규칙 기반 복원력
- `conjugate()`의 생성 방향 검증
- `LRNounExtractor_v2`의 pragmatic한 예외 처리
- `NewsPOSExtractor`의 staged matching 철학

### 버릴 것

- 한 클래스가 데이터 적재부터 결과 저장까지 전부 맡는 구조
- mutable graph 소비가 외부에 잘 드러나지 않는 방식
- dict/namedtuple/raw internal state가 뒤섞인 결과 타입
- historical experiment와 mainline API가 같은 층에 놓인 패키지 구성

## 8.11 마이그레이션 전략

한 번에 갈아엎으면 위험하다. 나는 아래 순서로 옮긴다.

### 1단계: 공통 엔진 분리

- `ObservationStore`
- `BoundaryGraph`
- `MorphResolver`

를 먼저 만들고 기존 코드가 이것들을 내부적으로 쓰게 바꾼다.

### 2단계: analyzer 래핑

기존 `LRNounExtractor_v2`, `PredicatorExtractor`, `NewsPOSExtractor`를 그대로 두되, 새 analyzer 인터페이스로 감싼다.

### 3단계: 결과 타입 통일

dict 반환을 유지하되 내부적으로는 `Candidate`, `Decision`, `Evidence` 객체를 만들고 마지막에 호환 변환한다.

### 4단계: compat 계층 도입

기존 import 경로를 유지하면서 내부 구현만 새 계층을 타게 만든다.

### 5단계: 신규 알고리즘 추가

이 시점에 `WordRank`, `KRWordRank`, keyword pipeline을 같은 구조에 올린다.

즉 이 재구성은 "현재 코드를 버리고 새 프로젝트를 만든다"가 아니라, 공통 엔진을 먼저 드러내는 방향의 점진적 재설계다.

## 8.12 이 구조가 주는 실제 이점

### 1. 설명 가능성이 올라간다

지금도 알고리즘은 설명 가능하지만, 코드 구조가 그걸 잘 보존하지 못한다. 새 구조에서는 후보마다 evidence와 decision log를 갖게 할 수 있다.

### 2. 실험 비용이 줄어든다

예를 들어 "KR-WordRank 스타일 R pruning을 noun analyzer에 섞어 보고 싶다" 같은 실험이 쉬워진다.

### 3. 자산 교체가 쉬워진다

현재는 feature 사전과 알고리즘이 강하게 결합돼 있다. 새 구조에서는 asset loader만 갈아 끼우면 된다.

### 4. 한국어 외 확장 가능성이 생긴다

strict한 형태 복원은 한국어 특화지만, boundary graph 자체는 다른 언어에도 확장 가능하다.

### 5. 문서 구조도 더 자연스러워진다

레포 구조 자체가 `관측 -> 그래프 -> 랭킹 -> 복원 -> 파이프라인`으로 정리되면, 지금 작성한 문서들도 훨씬 자연스럽게 레포를 따라가게 된다.

## 8.13 이 제안의 위험과 단점

좋은 점만 있는 건 아니다.

### 1. 재구성 비용이 크다

특히 `noun`과 `predicator`는 오랜 기간 예외 규칙이 쌓인 코드라, 구조를 바꾸는 순간 사소한 회귀가 많이 날 수 있다.

### 2. 너무 추상화하면 연구 속도가 느려질 수 있다

현재 코드는 거칠지만 실험 아이디어를 빨리 넣기 쉽다. 새 구조는 그 민첩성을 일부 잃을 수 있다.

### 3. typed graph 설계가 과하면 오히려 복잡해질 수 있다

처음부터 모든 타입을 정교하게 모델링하려 하면 구현이 무거워진다. 처음에는 `L`, `R`, `surface`, `lemma`, `feature` 정도의 작은 타입 체계로 시작하는 편이 낫다.

즉 이 재설계는 "가장 아름다운 구조"가 아니라 "현재 알고리즘의 강점을 해치지 않으면서 공통 엔진을 드러내는 구조"를 목표로 해야 한다.

## 8.14 내가 실제로 첫 번째로 바꿀 파일들

만약 정말 손대기 시작한다면 순서는 아래와 같이 간다.

1. `soynlp/utils/utils.py`
   `EojeolCounter`, `LRGraph`를 `core`/`graph` 계층으로 분리
2. `soynlp/lemmatizer/_lemmatizer.py`, `soynlp/lemmatizer/_conjugation.py`
   morphology engine으로 승격
3. `soynlp/noun/_noun_ver2.py`
   analyzer 조각으로 분해
4. `soynlp/predicator/_eomi.py`, `soynlp/predicator/_stem.py`
   독립 analyzer화
5. `soynlp/predicator/_predicator.py`
   pipeline coordinator로 축소
6. `soynlp/pos/_news_pos.py`
   staged pass pipeline으로 재구성

즉 첫 번째 타겟은 겉보기 기능보다 공통 엔진과 가장 중심적인 analyzer들이다.

## 8.15 최종 제안

내 최종 제안은 간단하다.

### 새 정체성

`soynlp`를 `boundary-aware lexical graph platform`으로 재정의한다.

### 새 중심축

- 관측 저장소
- 경계 그래프
- 형태 복원
- analyzer 조립

### 새 계보

- WordRank: untyped graph ranking
- KR-WordRank: typed graph ranking
- Noun extraction: feature-seeded typed graph classification
- Predicator extraction: graph + morphology resolution
- POS extraction: staged analyzer orchestration

### 새 운영 방식

- 내부는 공통 엔진 중심
- 외부는 compat API 유지
- 결과는 evidence-driven object로 통일
- 그래프 소비는 explicit state transition으로 기록

이렇게 가면 이 레포는 단지 "오래된 한국어 NLP 도구 모음"이 아니라, 한국어 비지도 어휘 추출 계열 알고리즘을 체계적으로 실험하고 확장할 수 있는 플랫폼으로 다시 태어날 수 있다.

## 8.16 이 문서를 어떻게 활용하면 좋은가

- 레포를 리팩터링할지 판단할 때는 8.3, 8.6, 8.11만 먼저 본다.
- 새 패키지 트리를 실제로 만들고 싶다면 8.6, 8.7, 8.14를 기준으로 작업을 쪼갠다.
- WordRank/KR-WordRank/soynlp의 관계를 설명해야 할 때는 8.5만 인용해도 큰 줄기가 잡힌다.
- 문서 세트를 더 발전시킬 때는 이 문서를 "미래 구조"의 기준점으로 두고, 현재 구조 문서와 대비해서 읽으면 된다.
