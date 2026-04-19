# 3. 알고리즘 워크스루

이 장은 패키지별 API 목록이 아니라, 실제로 이 레포가 어떤 방식으로 텍스트를 이해하려 하는지 "동작 순서" 중심으로 설명한다.

## 3.1 문자 계층: `hangle`

`soynlp.hangle`은 모든 상위 모듈의 바닥이다.

### 핵심 함수

- `compose(chosung, jungsung, jongsung)`: 자모를 완성형 음절로 조합
- `decompose(c)`: 완성형/자모를 `(초성, 중성, 종성)`으로 분해
- `character_is_korean`, `character_is_jaum`, `character_is_moum`: 문자 종류 판별
- `to_base(c)`: 문자 코드를 정수값으로 변환

### 왜 중요한가

이 레포는 불규칙 활용을 처리할 때 종성/모음 변화를 직접 계산한다. 예를 들어 `파랬다`를 `파랗 + 았다`로 복원하거나, `줬다`를 `주 + 었다`로 복원할 수 있는 이유는 결국 음절 내부 자모 구조를 알고 있기 때문이다.

### `ConvolutionHangleEncoder`

이 클래스는 자모와 숫자, 공백을 one-hot 인코딩하는 도구다. 프로젝트의 중심선은 아니지만 다음 역할을 한다.

- 한글을 초/중/종성 수준으로 펼친다.
- 숫자와 공백을 별도 차원으로 표현한다.
- 문자열을 신경망 입력 형태로 바꾸는 보조 인코더 역할을 한다.

### 거리 함수

`_distance.py`에는 다음 함수가 있다.

- `levenshtein`
- `jamo_levenshtein`
- `cosine_distance`
- `jaccard_distance`

특히 `jamo_levenshtein`은 한글 음절 전체를 1글자로 보지 않고, 분해된 자모 차이를 1/3 단위로 반영한다는 점이 특징이다.

## 3.2 정규화 계층: `normalizer`

`soynlp.normalizer`는 인터넷 텍스트, 댓글, 대화체처럼 노이즈가 많은 입력을 정리하는 도구다.

### `normalize`

허용할 문자군을 선택적으로 남긴다.

- 한글
- 영문
- 숫자
- 문장부호
- 괄호/기호류

그리고 반복 문자를 축약한다.

### `repeat_normalize`

- `ㅋㅋㅋㅋㅋㅋ` 같은 반복을 `ㅋㅋ` 같은 고정 길이로 줄인다.

### `emoticon_normalize`

가장 레포답게 흥미로운 함수다. 자음/모음 이모티콘과 완성형 한글이 엉켜 생긴 형태를 다시 분해한다.

예:

- `ㅋ쿠ㅜ` 같은 패턴을 자음/모음 반복으로 복원
- 완성형 속 종성을 분리해 웃음/울음 패턴을 정규화

### `only_hangle`, `only_hangle_number`, `only_text`

말 그대로 허용 문자만 남기는 필터다.

### `normalize_sent_for_lrgraph`

명사/용언 추출기의 전처리 핵심 함수다. LR graph를 만들기 전에 어절 끝에 붙은 불필요한 기호를 제거하고, 마지막 한글까지만 남기도록 도와준다.

## 3.3 토크나이저 계층: `tokenizer`

`soynlp.tokenizer`는 레포 안에서도 성격이 다른 토크나이저를 여럿 담고 있다.

### 3.3.1 `RegexTokenizer`

가장 단순하다. 아래 종류가 바뀌는 경계를 잘라낸다.

- 숫자
- 한글
- 자음
- 모음
- 영어/라틴 문자

즉 통계 기반 모델이라기보다 규칙 기반 토크나이저다.

### 3.3.2 `LTokenizer`

가정:

- 띄어쓰기가 어느 정도 맞다.
- 각 어절은 대체로 `L + optional R` 구조다.

동작:

1. 어절의 가능한 prefix를 모두 후보로 만든다.
2. 각 prefix에 점수를 붙인다.
3. 점수가 가장 높고, 동률이면 더 긴 L을 선택한다.
4. 나머지를 R로 둔다.

입력 점수는 보통 `WordExtractor`의 cohesion 점수나 `LRNounExtractor_v2`의 noun score다.

### 3.3.3 `MaxScoreTokenizer`

가정:

- 띄어쓰기가 틀렸을 수 있다.
- 사람은 더 익숙한 단어를 먼저 눈에 띄게 본다.

동작:

1. 문자열 내부의 가능한 모든 substring을 스캔한다.
2. substring마다 점수와 길이를 붙인다.
3. 점수가 높은 후보부터 greedy하게 선택한다.
4. 겹치는 후보를 제거한다.
5. 비어 있는 틈은 다시 작은 subtoken으로 채운다.

즉 "전체 문자열에서 높은 점수의 조각을 먼저 박아 넣는" 방식이다.

### 3.3.4 `MaxLRScoreTokenizer`

이 클래스는 `L` 사전과 `R` 사전을 함께 쓰는 좀 더 공격적인 분해기다.

핵심 재료:

- `Dl`: 왼쪽 단어 사전/점수
- `Dr`: 오른쪽 단어 사전/점수
- `lrgraph`: 실제 코퍼스 L-R 연결 확률
- `preference_l`, `preference_r`: 사용자 선호 가중치

동작 개요:

1. 가능한 L 후보를 찾는다.
2. 각 L에 대해 가능한 R을 확장한다.
3. L 점수, R 점수, 길이, 겹침 여부를 종합해 점수화한다.
4. 서로 겹치지 않는 최적 LR 세트를 고른다.
5. 빈 구간은 base tokenizer로 채운다.

코드상으로는 흥미로운 실험작이지만, 현재 트리 기준으로는 주변 의존성과 일부 속성명이 오래된 흔적을 보인다. 그래서 "중심 경로"보다는 "실험적 확장선"으로 읽는 편이 맞다.

### 3.3.5 명사 전용 토크나이저

`_noun_tokenizer.py`에는 두 가지가 있다.

- `NounLMatchTokenizer`: 문자열 매칭으로 명사만 왼쪽부터 붙여 나감
- `NounMatchTokenizer`: `MaxScoreTokenizer` 기반으로 명사 점수만 사용

즉 일반 토크나이저보다 "명사 조합을 읽어내는" 데 특화된 경량 도구다.

### 3.3.6 `EojeolPatternTrainer`

이 클래스는 LR graph를 만드는 훈련기다.

역할:

- substring vocabulary를 스캔
- `lrgraph`, `rlgraph` 생성
- 파일 저장/로드
- graph rank와 비슷한 `train_hits` 실험 함수 제공

지금의 중심 코드에서는 직접 많이 호출되지는 않지만, LR graph 기반 접근을 독립된 훈련기로 빼 보려던 흔적을 보여준다.

## 3.4 단어 점수화 계층: `word`

### 3.4.1 `WordExtractor`

이 클래스는 soynlp를 대표하는 모듈 중 하나다.

핵심 아이디어:

- 단어 후보의 왼쪽 응집도(cohesion)
- 오른쪽 응집도
- 좌/우 branching entropy
- 좌/우 accessor variety

를 함께 계산해 "단어처럼 보이는 substring"을 찾는다.

### 내부 상태

- `L`: prefix 빈도
- `R`: suffix 빈도
- `_aL`, `_aR`: 문맥과 결합된 accessor/branching 계산용 보조 카운터

### `train()`

1. 각 어절의 prefix, suffix 빈도를 센다.
2. 이웃 어절과의 결합 패턴을 `_aL`, `_aR`에 기록한다.
3. 필요하면 중간 pruning을 한다.

### `extract()`

1. 모든 단어 후보에 대해 점수 묶음(`Scores`)을 계산한다.
2. 최소 cohesion/entropy/accessor/frequency 조건으로 걸러낸다.
3. `remove_subwords=True`면 긴 단어에 흡수되는 subword를 제거한다.

### Cohesion

`cohesion_score(word)`는 대략 "첫 글자 빈도 대비 현재 문자열 빈도가 얼마나 강하게 묶여 있는가"를 본다.

### Branching Entropy / Accessor Variety

이 프로젝트의 통계 감각이 잘 드러나는 부분이다.

- substring 양쪽에 어떤 글자/단어가 다양하게 붙는가
- 그 분포가 얼마나 균등한가

실제 단어는 문맥 다양성이 높고, 어중간한 내부 조각은 특정 맥락에만 붙는 경향이 있다는 점을 이용한다.

### 3.4.2 `Bigram`

두 단어 연속 패턴을 추출하는 작은 도구다.

지원 점수:

- 빈도
- PMI
- Mikolov 스타일 점수

프로젝트의 주력 축은 아니지만, 간단한 n-gram 추출기로 바로 쓸 수 있다.

### 3.4.3 `pmi`, `pmi_memory_friendly`

co-occurrence matrix를 PPMI로 바꾸는 함수다.

- 입력: `(word, context)` 희소행렬
- 출력: PMI가 threshold 이상인 항목만 남긴 희소행렬
- `alpha`, `beta`: smoothing 파라미터

`pmi_memory_friendly`는 메모리를 아끼기 위해 `dok_matrix`를 순차적으로 채우는 구현이다.

## 3.5 명사 추출 계층: `noun`

이 패키지는 역사적으로 여러 버전이 공존한다.

### 3.5.1 `LRNounExtractor` v1

이 버전의 핵심은 "오른쪽 feature(r)가 조사처럼 보이면, 왼쪽 substring은 명사일 가능성이 높다"는 아이디어다.

동작:

1. substring vocabulary를 스캔한다.
2. `lrgraph`를 만든다.
3. 각 후보 noun에 대해 오른쪽 feature를 모은다.
4. 학습된 `r -> coefficient` 사전을 사용해 명사 점수를 계산한다.
5. feature 수가 너무 적으면 subword score로 보완한다.
6. 후처리로 `Noun + Josa` 꼴을 제거한다.

반환형:

- `NounScore_v1(frequency, score, known_r_ratio)`

### 3.5.2 `NewsNounExtractor`

이 버전은 뉴스 도메인에 맞춘 규칙이 더 많이 들어가 있다.

특징:

- `r_scores`를 이용한 기본 점수화
- 뉴스 명사 사전/용언 사전과 함께 사용
- `_is_NJsubJ`, `_is_NVsubE`, `_is_compound`, `_hardrule_*` 같은 규칙 기반 후처리 다수
- `feature_proportion`, `eojeol_proportion`, `josa variety` 같은 세부 통계 사용

다만 현재 소스 트리 기준으로는 constructor 내부의 변수 참조와 일부 스타일이 오래된 흔적을 보여, "역사적 버전"으로 이해하는 편이 안전하다.

### 3.5.3 `LRNounExtractor_v2`

현재 레포에서 가장 중요한 명사 추출기다.

#### 기본 자원

이 클래스는 `trained_models/noun_predictor_ver2_pos`, `..._neg`를 읽어 다음 세 feature 집합을 만든다.

- `_pos_features`: 조사처럼 명사를 지지하는 오른쪽 feature
- `_neg_features`: 어미처럼 명사를 깎는 오른쪽 feature
- `_common_features`: 양쪽에 공통으로 나타나는 feature

#### 학습 입력

세 가지를 받을 수 있다.

- `sentences`
- `EojeolCounter`
- `LRGraph`

즉 상위 단계에서 미리 세 둔 통계를 그대로 재사용할 수 있다.

#### 명사 후보 생성

`_noun_candidates_from_positive_features()`는 양의 feature 뒤에서 관찰되는 모든 L을 후보로 모은다. 즉 "조사 비슷한 것이 붙은 왼쪽 조각"만 먼저 보는 셈이다.

#### 점수 계산

`predict(word)`는 `lrgraph.get_r(word)`로 오른쪽 feature를 읽은 뒤 아래를 계산한다.

- `pos`
- `common`
- `neg`
- `unk`
- `end`

그리고

- `score = (pos - neg) / (pos + neg)`
- `support = score 방향에 따라 선택된 지지 빈도`

를 만든다.

#### iterative batch prediction

`_batch_predicting_nouns()`가 이 클래스의 핵심이다.

1. 긴 후보부터 순회한다.
2. 후보를 명사로 예측한다.
3. score가 기준 이상이면 그 후보가 포함된 어절 패턴을 LR graph에서 제거한다.

이렇게 하면 이미 설명된 긴 명사가 짧은 노이즈 substring을 덮어주므로, 이후 후보 판단이 쉬워진다.

#### 복합명사 추출

`extract_compounds()`는 `MaxScoreTokenizer`를 이용해 긴 어절을 이미 찾은 명사들로 분해해 본다.

조건:

- 마지막 토큰이 조사면 `Noun* + Josa`
- 아니면 모든 토큰이 명사여야 함

그리고 `_compounds_components`에 분해 결과를 따로 저장한다.

#### 후처리

기본 후처리 체인은 다음 세 가지다.

- `detaching_features`: `명사 + feature`가 붙은 후보 제거
- `ignore_features`: feature 자체와 같은 표면형 제거
- `check_N_is_NJ`: 사실상 `Noun + Josa`로 더 잘 설명되는 후보 제거

#### domain pos feature 추출

`extract_domain_pos_features()`는 현재 코퍼스에서 새 조사/양의 feature를 더 뽑아내는 보조 루틴이다. `_josa.py`가 이를 담당한다.

이 기능은 "고정된 조사 목록에만 기대지 말고, 현재 도메인에서 자주 붙는 오른쪽 기능형을 추가로 발견하자"는 목적을 가진다.

## 3.6 활용형 생성과 역복원: `lemmatizer`

`lemmatizer`는 이 레포의 규칙 엔진이다.

### 3.6.1 `conjugate`

stem + ending을 받아 가능한 표면형을 만든다.

지원 규칙은 매우 많다. 대표적으로:

- ㄷ 불규칙
- 르 불규칙
- ㅂ 불규칙
- 어미 첫 글자가 자음인 경우
- ㅅ 불규칙
- 우/오 활용
- ㅡ 탈락
- 거라/너라
- 러 불규칙
- 여 불규칙
- ㅎ 탈락/축약
- 이 + 어 -> 여

`conjugate_chat`은 여기에 채팅체/초성형 endings를 더 허용하는 확장 버전이다.

### 3.6.2 `_conjugate_stem`

어간만으로 가능한 표면 stem 부분들을 생성한다. `PredicatorExtractor`, `EomiExtractor`, `StemExtractor`가 모두 이 함수를 써서 "기본형 stem이 코퍼스에서 어떤 표면형으로 나타날 수 있는가"를 계산한다.

### 3.6.3 `lemma_candidate`

표면형을 `(stem, eomi)` 후보들로 거꾸로 복원하는 함수다.

핵심 구조:

1. 현재 `l`, `r` 경계를 기준으로 기본 후보 `(l, r)`를 둔다.
2. 음절 내부 자모를 분석한다.
3. 불규칙 활용을 역으로 되돌리는 여러 규칙을 적용한다.
4. 각 후보에 대해 다시 `conjugate(stem, eomi)`를 호출해 원래 표면형이 실제로 생성되는지 검증한다.

즉 단순 규칙 나열이 아니라 "역복원 후보 생성 -> 재활용 검증" 구조다.

### 3.6.4 `Lemmatizer`

이 클래스는 위 함수를 감싼 고수준 인터페이스다.

- stem 집합
- ending 집합

을 알고 있을 때 어떤 표면형이 가능한 기본형 후보로 복원되는지 알려준다.

## 3.7 용언 추출 계층: `predicator`

### 3.7.1 왜 별도 패키지인가

명사는 조사 특징만으로 어느 정도 접근할 수 있지만, 용언은 훨씬 어렵다.

- stem과 ending이 결합하며 표면형이 크게 바뀜
- 불규칙 활용 존재
- 명사와 표면형이 겹치는 경우 많음

그래서 이 레포는 명사 추출 이후 별도 패키지에서 용언을 다룬다.

### 3.7.2 `PredicatorExtractor`

이 패키지의 중심 클래스다.

#### 초기 자원

- 기본 josa
- 기본 adjective/verb stems
- 기본 eomis

를 사전 파일에서 읽어온다.

#### 사전 정리

`_remove_stem_prefix()`는 이미 noun으로 설명되는 stem prefix를 제거해, 명사와 용언 후보 충돌을 줄인다.

#### 표면 stem 계산

`_transform_stem_as_surfaces()`는 모든 stem에 대해 `_conjugate_stem()`을 적용해 가능한 stem surface를 만든다.

#### predicator 전용 LR graph 준비

`_prepare_predicator_lrgraph()`는 noun이 포함된 어절을 제거한 뒤 LR graph를 다시 만든다. 즉 명사 단계가 덮은 영역을 빼고 남은 어절에서 용언 패턴을 본다.

#### eomi 추출

`extract_eomi=True`면 `EomiExtractor`를 호출한다.

#### stem 추출

`extract_stem=True`면 `StemExtractor`를 호출한다.

#### 최종 lemmatization

`_as_lemma_candidates()`는 각 eojeol의 가능한 `(stem, eomi)` 조합을 모두 찾고, 실제 conjugation으로 검증해 predicator를 만든다.

#### 형용사/동사 분리

`_separate_adjective_verb()`는 아래 순서로 분리한다.

1. 기존 adjective/verb 사전 우선
2. suffix 기반 규칙 분류
3. 현재형/명령형/청유형 활용 테스트

즉 `PredicatorExtractor`의 최종 출력은 "용언"만이 아니라 "형용사 집합"과 "동사 집합"이다.

### 3.7.3 `EomiExtractor`

핵심 아이디어:

- known stem surface 뒤에 붙는 R들을 모으면 어미 후보가 된다.
- noun과 혼동되는 경우, 더 긴 stem surface가 존재하는 경우를 제외한다.

과정:

1. stem surface에서 관측된 R 후보 수집
2. 후보마다 `predict()`로 pos/neg/unk 평가
3. 양성 후보를 다시 `lemma_candidate()`로 기본형 eomi에 묶음

즉 표면형 ending을 바로 끝내지 않고 canonical eomi로 환원한다.

### 3.7.4 `StemExtractor`

핵심 아이디어:

- known eomi 뒤에서 관찰되는 L들을 모으면 stem 후보가 된다.
- 다양한 R 문자로 확장되는지, entropy가 충분한지 본다.

과정:

1. known stems/eomis의 표면형 집합 `L`, `R` 구성
2. R 후보를 기준으로 새 L 후보 수집
3. 양/음/미지 feature 기반 점수 계산
4. `lemma_candidate()`로 canonical stem에 묶음

즉 eomi 추출과 stem 추출은 서로 대칭적인 구조를 가진다.

## 3.8 코퍼스 기반 품사 추출: `pos`

### 3.8.1 `NewsPOSExtractor`

이 클래스는 "명사와 용언을 먼저 찾은 뒤, 남은 eojeol을 여러 패턴으로 덧분류해 품사 bucket을 만든다"는 구조다.

전체 흐름:

1. `LRNounExtractor_v2`로 명사 추출
2. 그 LR graph를 바탕으로 `PredicatorExtractor` 실행
3. 기본 adverb/josa/eomi/stem 자원 결합
4. 남은 eojeol을 패턴별로 차례로 소비

세부 단계는 `_count_matched_patterns()`에 거의 그대로 드러난다.

- `_match_word`: eojeol이 이미 noun/adjective/verb/adverb 자체인 경우 매칭
- `_match_noun_and_word`: `Noun + Josa/Adjective/Verb` 분리
- `_match_predicator_compounds`: 용언 복합어 처리
- `_lemmatizing_predicators`: 남은 표면형을 lemmatize해 용언 추가
- `_match_syllable_noun_and_r`: 1음절 명사 + R 보정
- `_remove_irregular_words`: 조사/어미 단독, 1음절 노이즈 제거
- `_match_compound_nouns`: 남은 eojeol에서 복합명사 추출

즉 이 클래스는 "확률 모델 하나"가 아니라, 여러 명시적 패턴을 순서대로 적용하는 orchestrator다.

### 3.8.2 `ChatPOSExtractor`

`NewsPOSExtractor`를 상속해 채팅체에 맞는 몇 가지 규칙을 추가한다.

차이점:

- 채팅용 josa 사전 사용
- noun phrase를 confused noun으로 처리
- predicator compound 처리에서 잘못된 stem/eomi 제거 규칙 추가
- 복합명사 매칭을 일부 비활성화

즉 뉴스 텍스트보다 더 흐트러진 대화체를 다루려는 변형이다.

## 3.9 사전 기반 품사 태깅: `postagger`

이 패키지는 `pos`와 성격이 다르다. `pos`가 코퍼스 통계로 품사를 "추출"한다면, `postagger`는 이미 알고 있는 품사 사전을 써서 문장을 "분해/태깅"한다.

### 3.9.1 `Dictionary`

아주 단순한 품사 사전 래퍼다.

- `get_pos(word)`
- `word_is_tag(word, tag)`
- `add_words()`
- `remove_words()`
- `save()`, `load()`

### 3.9.2 `EojeolTemplateMatcher`, `LRTemplateMatcher`

두 matcher는 후보를 생성한다.

- `EojeolTemplateMatcher`: 전체 어절 단위 후보 + compound noun/adverb 분해
- `LRTemplateMatcher`: 가능한 L 후보를 찾고, 허용된 품사 템플릿에 맞는 R을 확장

즉 태거의 generator 역할이다.

### 3.9.3 Evaluator들

- `SimpleEojeolEvaluator`
- `LREvaluator`

이들은 후보 분해들에 점수를 붙인다.

주로 보는 요소:

- noun 여부
- known LR 여부
- 길이
- unknown 포함 여부
- compound noun 여부
- 사용자 preference

### 3.9.4 `SimpleTagger`

구조는 전형적이다.

1. matcher가 후보 생성
2. evaluator가 최고 후보 선택
3. postprocessor가 미인식 조각 보정
4. 필요하면 flatten

### 3.9.5 `UnknowLRPostprocessor`

선택된 후보 사이의 빈 구간을 `None` 태그 subword로 메운다.

즉 태거가 모르는 부분을 버리지 않고 그대로 보존한다.

### 3.9.6 `_lrtagger.py`

이 파일은 LR graph, cohesion, droprate까지 사용해 더 고급 태거를 만들려는 실험선이다. 구조 자체는 흥미롭지만, 현재 코드 트리 기준으로는 `Dictionary`의 현재 구현과 완전히 맞물리지는 않는다. 그래서 "실험적/과거 경로"로 보는 편이 좋다.

## 3.10 벡터화와 연관어 계산: `vectorizer`

### `BaseVectorizer`

역할:

- tokenizer를 이용해 bag-of-words sparse matrix 생성
- min/max tf, df 기준 필터링
- vocabulary 저장/로드
- 문서 단위 list/bow 인코딩/디코딩

특징:

- 메모리에 직접 올리는 `fit_transform`
- MatrixMarket 형식 파일로 바로 쓰는 `fit_to_file`, `to_file`

### `sent_to_word_contexts_matrix`

이 함수는 word-context co-occurrence 행렬을 만든다.

핵심 옵션:

- `windows`
- `min_tf`
- `tokenizer`
- `dynamic_weight`

`word/_pmi.py`와 함께 쓰면 연관어 분석 파이프라인이 완성된다.

## 3.11 기타 보조 모듈

### `pos/_adverb.py`

- 기본 adverb 사전 로드
- `-하` stem을 `-히` 부사로 바꾸는 간단한 규칙 제공

### `utils/math.py`

- `randomized_svd`를 감싼 thin wrapper

### `ner/_rules.py`

- 날짜/요일/숫자/금액/기간/시간 판별 함수 자리만 있음
- 현재는 `raise NotImplemented`

즉 NER은 아직 실질 기능이 없는 스텁이다.

## 3.12 이 장에서 꼭 기억할 것

- `WordExtractor`는 점수 학습기, `Tokenizer`는 그 점수를 소비하는 분해기다.
- `LRNounExtractor_v2`는 이 레포의 현재 대표 명사 추출기다.
- `lemmatizer`는 stem/eomi를 오가는 규칙 엔진이다.
- `PredicatorExtractor`는 명사가 덮지 못한 영역에서 용언을 복원한다.
- `NewsPOSExtractor`는 여러 패턴을 순서대로 적용하는 상위 orchestration 레이어다.
- `postagger`는 별개의 "사전 기반" 태거 라인이다.

