# 7. 구현 상세 가이드

이 문서는 앞선 장들이 설명한 개념을 실제 코드 구현과 직접 연결해 주는 문서다. 목표는 "무슨 아이디어를 썼는가"에서 멈추지 않고, "그 아이디어가 실제로 어떤 클래스, 어떤 필드, 어떤 메서드 호출 순서로 구현되어 있는가"를 따라가게 만드는 것이다.

이 장은 특히 아래 파일들을 중심으로 읽는다.

- `soynlp/utils/utils.py`
- `soynlp/word/_word.py`
- `soynlp/tokenizer/_tokenizer.py`
- `soynlp/noun/_noun_ver2.py`
- `soynlp/lemmatizer/_lemmatizer.py`
- `soynlp/lemmatizer/_conjugation.py`
- `soynlp/predicator/_predicator.py`
- `soynlp/predicator/_eomi.py`
- `soynlp/predicator/_stem.py`
- `soynlp/pos/_news_pos.py`

## 7.1 먼저 잡아야 하는 구현 관점

이 레포는 큰 모델 하나가 전부 해결하는 구조가 아니다. 대신 아래 순서로 여러 단계가 이어진다.

```text
문장 목록
-> EojeolCounter
-> LRGraph
-> noun / predicator / pos 추출기
-> 사전형 또는 표면형 결과 묶음
```

핵심은 대부분의 추출기가 "정적 그래프를 한 번 읽고 끝내는 것"이 아니라, `LRGraph`를 점진적으로 깎아 가며 다음 단계를 위한 잔여 패턴을 남긴다는 점이다.

즉 실제 구현을 읽을 때는 다음 세 가지를 계속 확인해야 한다.

- 지금 이 클래스가 읽는 입력은 `sentences`, `EojeolCounter`, `LRGraph` 중 무엇인가
- 지금 이 메서드는 `self.lrgraph`를 읽기만 하는가, 아니면 `remove_eojeol()`로 상태를 바꾸는가
- 지금 저장되는 결과가 표면형 빈도인가, `(stem, eomi)` 같은 기본형 후보인가

## 7.2 공통 기반 구현

### `DoublespaceLineCorpus`

이 클래스는 `utils.py`에서 말뭉치를 지연 로딩하는 가장 얇은 입구다.

- 생성자에서 파일 경로, 문서 수 제한, 문장 수 제한, `iter_sent`, `skip_header`를 받는다.
- `_check_length()`는 파일을 한 번 훑어 `num_doc`, `num_sent`를 계산한다.
- `__iter__()`는 `iter_sent=False`면 문서 단위로, `iter_sent=True`면 `'  '` 두 칸 공백을 기준으로 문장 단위로 나눠 내보낸다.
- 즉 이 클래스의 핵심 책임은 "파일을 한 번에 다 메모리에 올리지 않고, soynlp가 기대하는 doublespace 형식으로 문장을 흘려보내는 것"이다.

구현상 중요한 점:

- `len(corpus)`가 필요하기 때문에 `check_corpus()`를 통과하려면 `__len__`이 동작해야 한다.
- `skip_header`는 iterator와 length 계산 둘 다에 적용된다.

### `EojeolCounter`

대부분의 학습 파이프라인은 결국 이 객체를 거친다.

핵심 필드:

- `self._counter`: `eojeol -> count`
- `self._count_sum`: 전체 eojeol 빈도 합
- `self._coverage`: 후속 단계에서 얼마나 많은 eojeol을 설명했는지 표시
- `self.preprocess`: 문장 단위 전처리 함수

실제 카운팅은 `_counting_from_sents()`가 담당한다.

```text
for sent in sents:
    sent = preprocess(sent)
    for eojeol in sent.split():
        길이 조건을 만족하면 카운트 증가
```

구현 디테일:

- `filtering_checkpoint`가 켜져 있으면 일정 문장 수마다 `min_count` 미만 항목을 중간 필터링한다.
- `max_length`를 넘는 eojeol은 처음부터 버린다.
- `remove_covered_eojeols()`는 결과만 따로 저장하는 게 아니라 실제 `_counter`를 줄여 가며 `coverage`를 갱신한다.

### `LRGraph`

이 레포의 진짜 공용 작업 메모리다.

핵심 필드:

- `self._lr`: `L -> {R: count}` 사전
- `self._rl`: `R -> {L: count}` 역방향 사전
- `self._lr_origin`: 원본 스냅샷

이 세 필드를 이해하면 왜 많은 추출기가 `reset_lrgraph()`를 호출하는지 바로 보인다.

구현 포인트:

- `EojeolCounter.to_lrgraph()`는 모든 eojeol을 가능한 prefix/suffix 분해로 넣는다.
- 이때 `r=''`도 저장하므로 "어절 자체가 끝나는 경우"를 바로 셀 수 있다.
- `remove_eojeol(eojeol, count)`는 eojeol 전체를 다시 모든 분할로 나눠 `_lr`, `_rl`에서 차감한다.
- `reset_lrgraph()`는 `_lr_origin`을 다시 복사해 원래 상태로 되돌린다.

즉 noun extractor나 eomi extractor가 어떤 후보를 확정할 때 `remove_eojeol()`을 호출하는 것은, 단순한 후처리가 아니라 "이미 설명된 패턴을 그래프에서 지워 다음 단계가 잔여 패턴만 보게 하려는 구현"이다.

추가로 알아둘 점:

- `copy_compatified_lrgraph_origin()`은 `_lr_origin`만 가진 호환용 복사본을 만든다.
- `to_EojeolCounter(reset_lrgraph=False)`는 현재 그래프 상태를 eojeol 카운터처럼 다시 바꿔 준다.

## 7.3 `WordExtractor` 구현

### 내부 상태

`soynlp/word/_word.py`의 `WordExtractor`는 아래 네 종류의 카운터를 쌓는다.

- `L`: 단어 prefix 빈도
- `R`: 단어 suffix 빈도
- `_aL`: 오른쪽 인접 정보
- `_aR`: 왼쪽 인접 정보

`train()`을 보면 실제로 이렇게 채운다.

1. 각 어절의 모든 prefix를 `L`에 누적
2. 각 어절의 모든 suffix를 `R`에 누적
3. 문장 안 인접 단어의 첫 글자/마지막 글자를 이용해 `_aL`, `_aR`를 누적

중요한 구현 디테일:

- `_aL`의 키는 `"word next_char"` 꼴이다.
- `_aR`의 키는 `"prev_char word"` 꼴이다.
- 코드가 `zip([words[-1]]+words[:-1], words, words[1:]+[words[0]])`를 쓰므로, 구현상 첫 단어의 왼쪽과 마지막 단어의 오른쪽은 같은 문장 안 반대쪽 단어와 감겨 연결된다.

### 점수 계산 흐름

실제 호출 순서는 아래다.

```text
train()
-> word_scores()
   -> all_cohesion_scores()
   -> all_branching_entropy()
   -> all_accessor_variety()
-> extract()
```

각 메서드의 구현 의미:

- `cohesion_score(word)`: `self.L[word] / self.L[first_char]`, `self.R[word] / self.R[last_char]`를 길이 보정한 거듭제곱으로 계산
- `branching_entropy(word)`: 한 글자 더 확장된 좌우 패턴의 엔트로피 계산
- `accessor_variety(word)`: 엔트로피 대신 가능한 확장 수만 센 버전
- `extract()`: 최소 빈도, 최소 cohesion, 최소 branching entropy 같은 threshold를 한 번에 적용

즉 `WordExtractor`는 "하나의 점수"를 바로 학습하는 구현이 아니라, 통계를 먼저 다 쌓아 두고 마지막에 조건을 조합하는 구조다.

## 7.4 tokenizer 구현

### `RegexTokenizer`

이 클래스는 가장 규칙 기반이다.

- `_patterns`에 숫자, 한글, 자음, 모음, 라틴 문자열 regex를 순서대로 둔다.
- `_tokenize()`는 패턴 하나를 적용해 찾은 문자열 주위를 공백으로 감싸고, 다시 다음 패턴을 적용한다.
- 마지막에 `\s+`를 하나로 줄여 토큰 리스트를 만든다.

즉 정교한 형태소 분석기가 아니라 "문자 종류 경계 위주로 거칠게 자르는 전처리용 구현"이다.

### `LTokenizer`

핵심 함수는 `tokenize()` 안쪽의 `token_to_lr()`다.

```text
모든 분할점 e에 대해 (token[:e], token[e:]) 후보 생성
-> 왼쪽 조각의 score 조회
-> 최고 점수 후보 선택
```

세부 동작:

- 길이 2 이하 어절은 그대로 `(token, '')`
- 후보는 길이 2 이상인 왼쪽 조각만 본다
- `tolerance > 0`이면 최고점 근처 후보만 남기고, 그중 더 긴 L을 선택한다
- `flatten=True`면 `(L, R)`를 펼쳐 리스트로 만든다

즉 실제 구현은 "오른쪽 점수를 따로 모델링"하지 않고, 왼쪽 후보 점수 하나만으로 L/R 분해를 정한다.

### `MaxScoreTokenizer`

이 클래스는 noun compound 분해에서 매우 중요하다.

토큰 하나를 자를 때 내부 표현은 `(subtoken, begin, end, score, length)` 튜플이다.

실제 흐름:

```text
_initialize()
-> 가능한 모든 부분 문자열 후보를 score와 함께 생성
-> 점수 내림차순, 길이 내림차순 정렬
_find()
-> 가장 높은 점수 후보를 하나 고른 뒤
-> 겹치는 후보를 전부 제거
_add_inter_subtokens(), _add_first_subtoken(), _add_last_subtoken()
-> 비어 있는 구간을 기본 점수 조각으로 채움
```

즉 이 구현은 DP가 아니라 "가장 점수 높은 부분 문자열을 먼저 고르고 겹침을 제거하는 greedy 분해기"다.

### `MaxLRScoreTokenizer`

이 클래스는 사전, 선호도, LRGraph를 함께 쓰는 실험선이다.

- `Dl`, `Dr`: 좌우 사전
- `Pl`, `Pr`: 선호 사전
- `lrgraph_norm`: LRGraph를 확률처럼 정규화한 형태

다만 레포 전체에서 가장 중심 경로는 `LTokenizer`, `MaxScoreTokenizer`, 명사 추출기 v2 쪽이므로 이 클래스는 보조적으로 읽으면 된다.

## 7.5 `LRNounExtractor_v2` 구현

### 생성자에서 하는 일

`__init__()`는 크게 두 가지를 한다.

1. 길이 제한, 후처리 옵션, normalization 플래그 같은 동작 파라미터 저장
2. `_load_predictor()`로 기본 feature 사전 적재

`_load_predictor()`는 `trained_models/noun_predictor_ver2_pos`, `..._neg`를 읽어 아래 세 집합을 만든다.

- `_pos_features`
- `_neg_features`
- `_common_features`

즉 v2 명사 추출기의 출발점은 "코퍼스에서 feature를 바로 학습"이 아니라 "기본 긍정/부정 feature 집합을 먼저 로딩"하는 것이다.

### 학습 단계

`train()`은 입력 타입에 따라 세 갈래로 갈린다.

```text
sentences -> _train_with_sentences()
EojeolCounter -> _train_with_eojeol_counter()
LRGraph -> _train_with_lrgraph()
```

`_train_with_sentences()`는 `EojeolCounter`를 만들고, 거기서 다시 `to_lrgraph()`를 호출한다. 그래서 실제 핵심 상태는 결국 `self.lrgraph` 하나로 귀결된다.

### 추출 단계 전체 흐름

실제 메서드 순서는 아래다.

```text
extract()
-> _noun_candidates_from_positive_features()
-> (선택) extract_domain_pos_features()
-> _batch_predicting_nouns()
-> (선택) extract_compounds()
-> _post_processing()
-> _check_covered_eojeols()
-> (선택) lrgraph.reset_lrgraph()
```

### 후보 생성

`_noun_candidates_from_positive_features()`는 구현상 매우 단순하다.

- 모든 positive feature `r`를 돈다
- `lrgraph.get_l(r, -1)`로 해당 feature 앞에 실제로 왔던 L을 모은다
- 그 L들의 누적 빈도를 noun candidate 빈도로 쓴다

즉 "조사처럼 보이는 오른쪽 조각 앞에서 관찰된 모든 왼쪽 조각"이 1차 후보다.

### `predict(word)`의 실제 점수 계산

`predict()`는 `lrgraph.get_r(word, -1)`로 오른쪽 feature 전체를 읽고 `_predict()`에 넘긴다.

`_predict()`는 오른쪽 조각을 다섯 범주로 나눈다.

- `pos`
- `common`
- `neg`
- `unk`
- `end`

그리고 점수는 아래처럼 계산한다.

```text
base = pos + neg
score = (pos - neg) / base
support = score가 높으면 pos + end + common
         아니면 neg + end + common
```

구현상 중요한 예외 처리:

- feature 수가 부족해도 `end` 비율이 높거나
- 서로 다른 첫 글자 feature가 여러 개 붙거나
- 대부분 단일 eojeol로 관찰되면

완전히 버리지 않고 보정 점수를 준다.

그래서 `predict()`는 단순 선형 분류기라기보다 "규칙이 많이 들어간 휴리스틱 점수 함수"에 가깝다.

### 왜 `LRGraph`를 지우는가

`_batch_predicting_nouns()`는 noun score가 threshold를 넘으면 그 noun이 왼쪽에 붙은 모든 `word+r` 패턴을 `remove_eojeol()`로 지운다.

이 동작이 중요한 이유:

- 이미 명사로 설명된 어절 패턴을 그래프에서 제거
- 뒤쪽 후보가 같은 빈도를 중복 소비하지 않게 함
- 이후 compound, predicator 단계가 잔여 패턴만 보게 함

즉 noun extractor는 "그래프를 읽고 결과만 만드는 것"이 아니라 "설명된 패턴을 그래프에서 소비하는 구현"이다.

### 복합명사 추출

`extract_compounds()`는 `MaxScoreTokenizer`를 내부 분해기로 쓴다.

흐름:

1. 이미 점수가 나온 단일 noun을 길이 점수 사전으로 만듦
2. 그 사전으로 `MaxScoreTokenizer`를 생성
3. 긴 후보를 greedy하게 분해
4. `_parse_compound()`로 진짜 복합명사인지 확인
5. 복합명사로 인정되면 substring 후보 빈도를 차감

즉 구현상 compound 추출은 별도 모델이 아니라 "이미 찾은 명사를 다시 tokenizer 점수로 활용하는 2차 루프"다.

## 7.6 lemmatizer와 conjugation 구현

### `lemma_candidate()`

`soynlp/lemmatizer/_lemmatizer.py`에서 가장 중요한 함수다.

입력은 `(l, r)` 분할 하나고, 출력은 가능한 `(stem, eomi)` 후보 집합이다.

구현은 거의 규칙 사전의 연쇄다.

- ㄷ 불규칙
- 르 불규칙
- ㅂ 불규칙
- 어미 첫 글자가 종성인 경우
- ㅅ 불규칙
- 우/오 불규칙
- ㅡ 탈락
- 여 불규칙
- ㅎ 탈락/축약
- `였` 계열 규칙
- predefined 예외

마지막 단계가 특히 중요하다.

```text
for stem, eomi in candidates:
    surfaces = conjugate(stem, eomi)
    if 원래 word가 surfaces 안에 있으면 채택
```

즉 이 함수는 규칙으로 후보를 많이 만들되, 마지막에는 다시 `conjugate()`로 재생성 검증을 한다.

### `Lemmatizer.lemmatize()`

이 메서드는 단어 전체를 가능한 모든 분할점으로 잘라 `lemma_candidate()`를 호출한다.

```text
for i in 1..len(word):
    l = word[:i]
    r = word[i:]
    lemma_candidate(l, r)
```

그리고 stem 사전, ending 사전 안에 실제로 존재하는 후보만 남긴다.

즉 lemmatizer 클래스는 규칙 엔진의 얇은 래퍼이고, 진짜 핵심 구현은 `lemma_candidate()`와 `conjugate()`다.

### `_conjugate_stem()`와 `conjugate()`

predicator 쪽 구현을 읽으려면 이 함수 둘이 왜 중요한지 알아야 한다.

- `_conjugate_stem(stem)`: stem 표면형 후보만 생성
- `conjugate(stem, ending)`: stem과 어미를 합쳐 실제 활용형 표면형 생성

predicator extractor는 기존 stem/eomi 사전에서 가능한 표면형을 먼저 만들어 두고, 실제 코퍼스에서 관찰된 문자열과 맞는지 검사하는 방식으로 동작한다.

## 7.7 `PredicatorExtractor` 구현

### 생성자에서 준비하는 상태

`PredicatorExtractor.__init__()`는 다음 상태를 만든다.

- `_josas`
- `_adjective_stems`
- `_verb_stems`
- `_stems`
- `_eomis`
- `_stem_surfaces`
- `_nouns`

핵심은 `_stem_surfaces = _transform_stem_as_surfaces()`다. 여기서 `_conjugate_stem()`을 모든 stem에 적용해 표면형 후보를 미리 만든다.

또 하나 중요한 구현은 `_remove_stem_prefix()`다.

- 어떤 stem의 앞부분이 이미 noun이면 그 noun을 noun 집합에서 제거한다.
- 즉 noun과 stem이 겹치는 경계를 미리 정리해 둔다.

### 학습 흐름

실제 호출 순서:

```text
train()
-> _train_with_sentences() 또는 _train_with_eojeol_counter()
-> (선택) _prepare_predicator_lrgraph()
-> (선택) _extract_eomi()
-> (선택) _extract_stem()
```

`_prepare_predicator_lrgraph()`는 noun이 포함된 eojeol을 제거한 뒤, 남은 eojeol만으로 새 LRGraph를 만든다. 즉 predicator 단계는 noun extractor가 설명하고 남긴 잔여 어절을 보는 구조다.

### 최종 용언 추출

`extract()`는 내부적으로 `_extract_predicator()`를 호출한다.

```text
extract()
-> _extract_predicator()
   -> _as_lemma_candidates()
   -> _remove_wrong_eomis()
-> _separate_adjective_verb()
```

#### `_as_lemma_candidates()`

이 메서드가 predicator 추출의 중심이다.

- eojeol 하나마다 가능한 모든 분할점을 돈다
- 각 `(l, r)`에 대해 `lemma_candidate(l, r)`를 호출한다
- stem이 `_stems` 안에 있고 eomi가 `_eomis` 안에 있는 후보만 남긴다
- 다시 `conjugate(stem, eomi)`로 원래 eojeol이 실제 생성되는지 검증한다

즉 predicator 추출은 noun extractor처럼 그래프 점수를 직접 매기기보다, "사전 + 활용 규칙 + 실제 코퍼스 표면형 일치"를 이용한 후보 검증에 가깝다.

#### `_remove_wrong_eomis()`

이 메서드는 짧은 eomi가 실은 2음절 명사에 더 가깝게 보이는 경우를 제거한다.

- eomi별로 어떤 word들이 그 eomi로 분석됐는지 모은다
- 길이 2 명사 비율이 너무 높으면 잘못된 eomi로 판단한다
- 해당 eomi를 `_eomis`에서 제거하거나, 그 eomi를 쓰는 lemma만 삭제한다

즉 predicator 파이프라인은 한번 뽑고 끝내지 않고, noun과의 충돌을 다시 검사한다.

#### `_separate_adjective_verb()`

마지막 분리는 세 층으로 이루어진다.

1. 기본 verb/adjective 사전 우선
2. `rule_classify()` 규칙 우선
3. 현재형/명령형/청유형 활용 테스트

그래서 이 단계는 순수 통계 분류기가 아니라 "사전 -> 규칙 -> 표면형 테스트" 순의 휴리스틱 체인이다.

## 7.8 `EomiExtractor`와 `StemExtractor`

### `EomiExtractor`

핵심 상태:

- `self._stems`
- `self._nouns`
- `self._stem_surfaces`

`extract()`의 실제 순서:

```text
extract()
-> _candidates_from_stem_surfaces()
-> _batch_prediction()
-> _eomi_lemmatize()
-> 빈도/점수 필터링
```

구현 포인트:

- `_candidates_from_stem_surfaces()`는 known stem surface 뒤에 실제로 관찰된 모든 R을 모은다.
- `predict(r)`는 `lrgraph.get_l(r, -1)`로 왼쪽 문맥을 보며
  - stem surface면 `pos`
  - noun+verb 꼴이면 `pos`
  - 뒤쪽 stem 흔적이 있으면 `unk`
  - 나머지는 `neg`
  로 센다.
- score가 높은 eomi는 다시 `_eomi_lemmatize()`에서 `lemma_candidate(stem_surface, eomi_surface)`를 통해 기본형 어미로 묶는다.

즉 이 구현은 "어미 표면형 후보 추출 -> 기본형 어미 복원"의 2단계다.

### `StemExtractor`

생성자에서 먼저 `_conjugate_stem_and_eomi()`를 돌려 다음 집합을 만든다.

- `self.L`: 기존 stem에서 만들어지는 표면형 L
- `self.R`: 기존 eomi와 결합해 실제 코퍼스에서 관찰된 표면형 R

즉 알려진 stem/eomi로부터 "이미 알고 있는 predicator 표면 공간"을 먼저 구축해 두는 셈이다.

`extract()`의 순서:

```text
R 후보를 기준으로 새 L 후보 수집
-> 빈도 필터링
-> _batch_prediction()
-> _post_processing()
-> _to_stem()
```

`predict(l)`는 단순 빈도만 보지 않는다.

- `lrgraph.get_r(l, -1)`의 R 분포
- 첫 글자 다양성
- 긍정 R의 엔트로피
- R이 조사인지, 이미 predicator로 분해 가능한지, 더 긴 eomi가 가능한지

를 함께 본다.

즉 stem extractor는 "오른쪽 확장 다양성"을 stem의 실재성 신호로 보는 구현이다.

## 7.9 `NewsPOSExtractor` 구현

이 파일은 여러 결과를 순서대로 조합하는 총괄기다. 구현을 이해할 때는 `train()`과 `extract()`를 분리해서 봐야 한다.

### 학습 단계

```text
train()
-> _train_noun_extractor()
-> _train_predicator_extractor()
-> 부사 사전 적재
-> 남은 eojeol 카운터 준비
```

`_train_noun_extractor()`는 `LRNounExtractor_v2`를 만들고 `extract(..., reset_lrgraph=False)`로 noun을 뽑는다. reset을 끄는 이유는 그 잔여 `lrgraph`를 뒤에서 predicator가 이어받아야 하기 때문이다.

`_train_predicator_extractor()`는 noun extractor에서 얻은 positive/common feature를 josa 집합처럼 넘겨 predicator extractor를 구성한다.

### 추출 단계

실제 핵심은 `_count_matched_patterns()`다.

```text
_match_word()
-> _match_noun_and_word()
-> _match_predicator_compounds()
-> _lemmatizing_predicators()
-> _match_syllable_noun_and_r()
-> _remove_irregular_words()
-> _match_compound_nouns()
```

각 패스의 역할:

- `_match_word()`: eojeol이 이미 noun/adjective/verb/adverb 자체인 경우 먼저 회수
- `_match_noun_and_word()`: `Noun + Josa`, `Noun + Adjective`, `Noun + Verb` 분리
- `_match_predicator_compounds()`: 기존 predicator 뒤에 다시 predicator가 붙는 복합 용언 탐지
- `_lemmatizing_predicators()`: 아직 안 잡힌 eojeol을 lemma 기반으로 다시 용언 판정
- `_match_syllable_noun_and_r()`: 1음절 명사 + 조사/용언 꼴 회수
- `_remove_irregular_words()`: 조사 단독, 어미 단독, 1음절 노이즈 제거
- `_match_compound_nouns()`: 남은 eojeol에서 복합명사 회수

즉 POS extractor는 "한 번의 통합 분류"가 아니라, 남은 eojeol 집합을 계속 줄여 가는 다단계 패턴 소거기다.

### 반환 구조

최종 `extract()`는 아래 딕셔너리를 만든다.

- `Noun`
- `Eomi`
- `Adjective`
- `AdjectiveStem`
- `Verb`
- `VerbStem`
- `Adverb`
- `Josa`
- `Irrecognizable`
- `ConfusedNouns`

여기서 `Adjective`, `Verb` 값은 단순 빈도 dict가 아니라 `Predicator(frequency, lemma)` 구조다. 반면 `Noun`, `Adverb`, `Josa`는 주로 카운터 성격의 dict다. 즉 반환 객체의 타입이 완전히 동일하지 않다는 점도 실제 구현을 읽을 때 알아둬야 한다.

## 7.10 실제 코드 읽기 순서

레포를 구현 수준으로 따라가려면 아래 순서가 가장 효율적이다.

1. `soynlp/utils/utils.py`
   `EojeolCounter`, `LRGraph`의 mutable 동작부터 잡아야 이후 코드가 보인다.
2. `soynlp/noun/_noun_ver2.py`
   이 레포의 대표적인 "graph를 소비하며 후보를 확정하는 구현"이다.
3. `soynlp/lemmatizer/_lemmatizer.py`
   용언 복원의 핵심 규칙이 여기 있다.
4. `soynlp/lemmatizer/_conjugation.py`
   `lemma_candidate()`가 왜 검증 가능한지 이해하려면 생성 방향도 봐야 한다.
5. `soynlp/predicator/_eomi.py`
   어미 추출이 stem surface와 LRGraph를 어떻게 결합하는지 보인다.
6. `soynlp/predicator/_stem.py`
   오른쪽 확장 다양성으로 stem을 판정하는 구현을 확인한다.
7. `soynlp/predicator/_predicator.py`
   noun/eomi/stem/lemma 후보를 어떻게 묶는지 본다.
8. `soynlp/pos/_news_pos.py`
   마지막으로 전체 파이프라인을 조합하는 상위 로직을 확인한다.

## 7.11 이 장에서 꼭 기억할 것

- 이 레포의 핵심 구현은 대형 모델이 아니라 mutable graph 위의 규칙적 소거와 재판정이다.
- `EojeolCounter`와 `LRGraph`는 단순 보조 자료구조가 아니라 거의 모든 추출기의 공용 작업 메모리다.
- noun extractor는 feature 점수와 예외 규칙을 섞은 휴리스틱 분류기다.
- predicator 계열은 `lemma_candidate()`와 `conjugate()`를 왕복 검증으로 사용한다.
- POS extractor는 단일 분류기가 아니라 여러 패스를 순차 적용하는 총괄 파이프라인이다.
