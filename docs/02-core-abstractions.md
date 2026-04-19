# 2. 핵심 추상화와 공통 개념

이 장은 `soynlp` 전반을 관통하는 자료구조와 용어를 정리한다. 이 레포의 거의 모든 주요 모듈은 아래 개념 위에 올라가 있다.

## 2.1 문장보다 자주 등장하는 기본 단위: 어절

이 레포는 "문장(sentence)"을 입력으로 받는 경우가 많지만, 실제 통계 계산은 대부분 `띄어쓰기 기준 단위`, 즉 어절(eojeol)을 중심으로 이루어진다.

예를 들어 문장 `"데이터마이닝을 공부한다"`는 아래처럼 다뤄진다.

- 문장 단위 입력: `"데이터마이닝을 공부한다"`
- 어절 단위 내부 처리: `"데이터마이닝을"`, `"공부한다"`

명사 추출기, 용언 추출기, POS 추출기는 거의 모두 어절 내부를 `L + R`로 자르는 관점에서 출발한다.

## 2.2 `L + R` 분해

이 레포의 핵심 추상화는 `L(left) + R(right)`다.

- `L`: 왼쪽 실질 의미 단위 후보
- `R`: 오른쪽 기능 단위 후보

보통 직관은 아래와 같다.

- `아이오아이의` -> `아이오아이` + `의`
- `먹었다` -> `먹` + `었다`
- `예쁘다` -> `예쁘` + `다`

물론 모든 어절이 깔끔하게 둘로만 나뉘는 것은 아니다. 그래서 실제 코드는 "가능한 모든 prefix/suffix 조합을 세고, 그중 통계적으로 가장 설명력이 좋은 조합"을 찾는다.

## 2.3 `DoublespaceLineCorpus`

파일 기반 말뭉치를 편하게 읽기 위해 `soynlp.utils.DoublespaceLineCorpus`가 제공된다.

이 클래스의 전제는 다음과 같다.

- 파일의 한 줄은 하나의 문서다.
- 한 문서 안의 문장 구분은 `\n`이 아니라 `"  "`(공백 두 칸)이다.
- `iter_sent=False`면 문서 단위, `iter_sent=True`면 문장 단위 iterator로 동작한다.

즉 이 레포에서 "문서형 corpus"와 "문장형 corpus"를 같은 파일 포맷으로 오갈 수 있게 해 주는 작은 어댑터라고 보면 된다.

## 2.4 `check_corpus`

여러 추출기는 공통적으로 `check_corpus()`를 호출한다. 이 함수가 강제하는 조건은 단순하다.

- `__iter__`가 있어야 한다.
- `__len__`이 있어야 한다.
- 길이가 0보다 커야 한다.

즉 이 레포는 generator 하나만 넘기기보다, 길이를 알 수 있는 corpus 객체나 list를 선호한다.

## 2.5 `EojeolCounter`

`EojeolCounter`는 이 레포의 실질적인 출발점이다.

역할은 다음과 같다.

- 문장 iterable을 받아 어절 빈도를 센다.
- 최소 빈도, 최대 길이, 중간 pruning checkpoint를 적용한다.
- `coverage`와 남은 미처리 어절 수를 추적한다.
- 자신을 `LRGraph`로 변환할 수 있다.
- 저장/로드가 가능하다.

특히 중요한 속성과 메서드는 아래와 같다.

- `_counter`: `{eojeol: count}` 딕셔너리
- `_count_sum`: 총 어절 빈도 합
- `coverage`: 얼마나 많은 어절을 상위 추출 단계가 덮었는지 기록
- `to_lrgraph()`: 어절 counter를 LR graph로 변환
- `remove_covered_eojeols()`: 덮인 어절 제거

`noun`, `predicator`는 어절을 바로 세기보다 먼저 `EojeolCounter`를 만든 뒤 여기에 기반해 LR graph를 생성하는 경우가 많다.

## 2.6 `LRGraph`

`LRGraph`는 이 레포의 핵심 자료구조다. 거의 모든 상위 알고리즘이 여기서 후보를 읽고, 제거하고, 다시 초기화한다.

구조는 개념적으로 아래와 같다.

- `_lr`: `{L: {R: count}}`
- `_rl`: `{R: {L: count}}`
- `_lr_origin`: reset을 위해 보관하는 원본 그래프

예를 들어 어절 `"아이오아이의"`가 있다면, 가능한 prefix/suffix 분해가 모두 추가된다.

- `아` + `이오아이의`
- `아이` + `오아이의`
- `아이오` + `아이의`
- `아이오아이` + `의`
- `아이오아이의` + `""`

이 구조가 중요한 이유는 다음과 같다.

- 명사 추출기는 `get_r(noun_candidate)`를 보며 오른쪽 feature를 관찰한다.
- 조사/어미 후보 추출은 `get_l(feature_candidate)`를 보며 왼쪽 문맥을 본다.
- 강한 명사를 찾은 뒤 `remove_eojeol(word + r, count)`로 해당 패턴을 지워 후속 추출을 쉽게 만든다.
- 추출이 끝나면 `reset_lrgraph()`로 다시 원본 상태로 되돌린다.

즉 LR graph는 단순한 "조회용 통계표"가 아니라, 여러 추출 단계가 공유하는 가변 작업 메모리다.

## 2.7 substring 빈도와 주변 문맥

이 레포는 단순 빈도만 보지 않는다. 대략 세 종류의 통계를 본다.

### 1. substring 빈도

- 특정 prefix가 얼마나 자주 등장하는가
- 특정 suffix가 얼마나 자주 등장하는가

이 정보는 `WordExtractor`의 `L`, `R`, `LRNounExtractor`의 substring counter, `EojeolCounter` 기반 LR graph 생성에 모두 쓰인다.

### 2. 경계 다양성

- 어떤 substring의 앞뒤에 다양한 글자/단어가 붙는가

이 정보는 branching entropy, accessor variety 같은 점수로 계산된다.

### 3. 기능형 feature와의 결합

- 어떤 L 뒤에 조사/어미 같은 양의 feature가 붙는가
- 어떤 L 뒤에 부정 feature나 미지 feature가 붙는가

이 정보는 명사/어미/어간 추출기의 핵심 분류 신호다.

## 2.8 공통 점수 구조

이 레포는 여러 namedtuple을 써서 점수를 반환한다. 가장 자주 등장하는 것들은 다음과 같다.

### `word.Scores`

`WordExtractor`가 반환하는 단어 점수 구조다.

- `cohesion_forward`
- `cohesion_backward`
- `left_branching_entropy`
- `right_branching_entropy`
- `left_accessor_variety`
- `right_accessor_variety`
- `leftside_frequency`
- `rightside_frequency`

### `noun.NounScore`

`LRNounExtractor_v2`가 반환하는 구조다.

- `frequency`
- `score`

여기서 `score`는 대체로 "긍정 feature 대비 부정 feature 우세도"를 나타낸다.

### `predicator.Predicator`

`PredicatorExtractor`, `NewsPOSExtractor`가 쓰는 구조다.

- `frequency`
- `lemma`

`lemma`는 하나 이상의 `(stem, eomi)` 후보 집합이다.

### `predicator.EomiScore`

`EomiExtractor`가 반환한다.

- `frequency`
- `score`

## 2.9 "빈도"와 "support"는 다르다

이 코드베이스에서는 어떤 후보를 평가할 때 단순 raw frequency만 쓰지 않는다. 자주 등장하는 추가 개념이 `support`다.

예를 들어 `LRNounExtractor_v2.predict()`는 대략 이런 구조다.

- `pos`: 긍정 feature가 붙은 횟수
- `neg`: 부정 feature가 붙은 횟수
- `common`: 긍정/부정 양쪽에 걸친 feature
- `unk`: 미지 feature
- `end`: 어절 끝에서 끝나는 빈도

그리고 최종적으로는

- 점수: `(pos - neg) / (pos + neg)`
- support: 점수 방향에 따라 신뢰 가능한 빈도를 다시 선택

즉 어떤 후보는 등장 횟수는 많아도 "명사로 지지하는 근거"는 약할 수 있다.

## 2.10 한글 처리 계층 두 개

이 레포에는 정규화 계층이 두 개 있다.

### `soynlp.hangle`

- 한글 음절/자모 수준 유틸리티
- `compose`, `decompose`
- 문자 판별 함수
- 예전 `normalize` 함수

### `soynlp.normalizer`

- 실제 텍스트 정규화용 함수
- 반복 문자 축약
- 이모티콘 분해
- `only_hangle`, `only_text`
- `normalize_sent_for_lrgraph`

즉 `hangle`은 문자 공학에 가깝고, `normalizer`는 말뭉치 전처리에 가깝다.

## 2.11 `normalize_sent_for_lrgraph`

여러 상위 모듈이 이 함수를 사용한다. 목적은 "어절 내부 통계용으로 너무 지저분한 기호를 제거하고, 마지막 한글까지만 남기자"는 것이다.

핵심 동작은 다음과 같다.

- 텍스트 필터로 한글/영문/숫자/일부 문장부호 외 제거
- 괄호류 같은 symbol 제거
- 각 어절에 대해 마지막 한글이 나타나는 위치까지만 남김
- 빈 어절은 버림

명사 추출기와 predicator 추출기가 깨끗한 LR graph를 만들 때 자주 쓴다.

## 2.12 `ensure_normalized` 플래그

여러 추출기 생성자에 `ensure_normalized`가 있는데, 이 이름은 "입력이 이미 정규화되었는가"라는 뜻으로 읽는 편이 맞다. 다만 현재 코드 트리 기준으로는 이 플래그의 적용 방식이 모듈마다 완전히 일관적이지 않다.

이 점은 [05-assets-tests-and-caveats.md](05-assets-tests-and-caveats.md)에서 별도 주의사항으로 다시 설명한다.

## 2.13 그래프를 지웠다가 되돌리는 패턴

이 레포를 이해할 때 매우 중요한 패턴이 있다.

1. 원본 LR graph를 만든다.
2. 강한 후보를 찾는다.
3. 그 후보가 덮는 `L+R` 조합을 graph에서 제거한다.
4. 제거된 graph 위에서 다음 후보를 더 잘 찾는다.
5. 필요하면 원본 graph로 `reset`한다.

이 패턴은 아래 모듈들에서 반복된다.

- `noun/_noun_ver2.py`
- `predicator/_eomi.py`
- `predicator/_predicator.py`
- 일부 POS 추출 흐름

그래서 `LRGraph._lr_origin`의 존재는 단순한 백업이 아니라 알고리즘 설계의 일부다.

## 2.14 표면형(surface)과 기본형(canonical form)

`lemmatizer`와 `predicator`에서 반복되는 구분이다.

- 표면형(surface): 코퍼스에 실제로 나타난 문자열
- 기본형(canonical form): 사전형 stem/eomi

예를 들어:

- 표면형: `줬다`
- 기본형 후보: `주 + 었다`

이 레포는 표면형을 바로 저장하지 않고, 가능한 한 `(stem, eomi)` 조합으로 환원하려고 한다. 그래서 활용형 생성(`conjugate`)과 역복원(`lemma_candidate`)이 둘 다 중요하다.

## 2.15 이 장에서 꼭 기억할 것

- 이 레포의 중심 단위는 token이 아니라 어절이다.
- 핵심 자료구조는 `EojeolCounter`와 `LRGraph`다.
- 많은 알고리즘은 `L + R` 분해를 기본 가정으로 한다.
- 점수는 단순 빈도보다 "어떤 feature와 어떻게 연결되는가"를 중시한다.
- 여러 추출기는 graph를 가변적으로 깎아 가며 동작한다.

