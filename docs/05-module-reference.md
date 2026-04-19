# 5. 모듈 레퍼런스

이 장은 `soynlp/` 소스 파일을 패키지별로 정리한 빠른 참조표다. "이 파일이 왜 존재하고, 무엇을 공개하며, 어떤 역할을 맡는가"를 짧고 촘촘하게 적었다.

## 5.1 루트 패키지

### `soynlp/__init__.py`

- 패키지 메타데이터를 정의한다.
- 하위 패키지 `hangle`, `normalizer`, `noun`, `predicator`, `postagger`, `tokenizer`, `vectorizer`, `word`, `utils`를 import해 공개한다.
- 하위 호환성 때문에 `DoublespaceLineCorpus`를 루트에서 다시 공개한다.

## 5.2 `soynlp.hangle`

### `soynlp/hangle/__init__.py`

- `levenshtein`, `jamo_levenshtein`, `cosine_distance`, `jaccard_distance`
- `normalize`, `compose`, `decompose`
- 문자 판별 함수와 `ConvolutionHangleEncoder`

### `soynlp/hangle/_hangle.py`

- 한글 유니코드 상수와 초/중/종성 테이블 정의.
- 예전 `normalize()` 함수 제공. 현재 README 기준으로는 `soynlp.normalizer` 쪽이 전처리용 중심이다.
- `compose`/`decompose`와 문자 판별 함수들의 실제 구현이 들어 있다.
- `ConvolutionHangleEncoder`가 한글을 자모/숫자/공백 one-hot 시퀀스로 바꾼다.

### `soynlp/hangle/_distance.py`

- 일반 Levenshtein 거리.
- 자모 분해 후 계산하는 `jamo_levenshtein`.
- 문자 multiset/set 기반 `cosine_distance`, `jaccard_distance`.

## 5.3 `soynlp.normalizer`

### `soynlp/normalizer/__init__.py`

- `normalize`, `emoticon_normalize`, `repeat_normalize`, `remove_doublespace`
- `only_hangle`, `only_hangle_number`, `only_text`
- `remain_hangle_on_last`, `normalize_sent_for_lrgraph`

### `soynlp/normalizer/_normalizer.py`

- regex 기반 전처리 함수 구현 파일.
- 반복문자 축약, 이모티콘 분해, 문자군 필터링을 제공한다.
- 상위 추출기들이 쓰는 `normalize_sent_for_lrgraph()`가 있다.

## 5.4 `soynlp.utils`

### `soynlp/utils/__init__.py`

- 메모리/파일/유사도 유틸과 `DoublespaceLineCorpus`, `EojeolCounter`, `LRGraph`, `svd`를 공개한다.

### `soynlp/utils/utils.py`

- 시스템 메모리 확인 함수 `get_available_memory`, `get_process_memory`.
- 파일 디렉터리 확인과 정렬 유틸.
- `most_similar()`로 벡터 공간상의 최근접 이웃 계산.
- `check_corpus()`로 iterable+len 제약 검증.
- `DoublespaceLineCorpus`, `EojeolCounter`, `LRGraph`의 실제 구현이 들어 있다.

### `soynlp/utils/math.py`

- `sklearn.utils.extmath.randomized_svd`를 감싼 `svd()` 하나만 제공한다.

## 5.5 `soynlp.word`

### `soynlp/word/__init__.py`

- `WordExtractor`, `pmi`, `Bigram`을 공개한다.

### `soynlp/word/_word.py`

- `Scores` namedtuple 정의.
- substring 응집도, branching entropy, accessor variety를 계산하는 `WordExtractor` 구현.
- 학습 결과를 `pickle`로 save/load하는 기능 포함.

### `soynlp/word/_pmi.py`

- co-occurrence 희소행렬을 PMI/PPMI로 바꾸는 함수.
- 일반 버전 `pmi`와 메모리 친화 버전 `pmi_memory_friendly` 제공.

### `soynlp/word/_phrase.py`

- bigram 추출기 `Bigram`.
- frequency, PMI, Mikolov 스타일 점수 중 하나를 사용해 bigram을 반환한다.

## 5.6 `soynlp.vectorizer`

### `soynlp/vectorizer/__init__.py`

- `BaseVectorizer`, `sent_to_word_contexts_matrix`를 공개한다.

### `soynlp/vectorizer/_vectorizer.py`

- tokenizer 기반 sparse term-frequency vectorizer.
- vocabulary 학습, sparse matrix 생성, MatrixMarket 형식 저장, bow/list 인코딩/디코딩 지원.

### `soynlp/vectorizer/_word_context.py`

- `(word, context)` co-occurrence 행렬 생성.
- vocabulary 스캔, 문맥 집계, CSR matrix 인코딩을 담당.

## 5.7 `soynlp.tokenizer`

### `soynlp/tokenizer/__init__.py`

- `LTokenizer`, `MaxScoreTokenizer`, `MaxLRScoreTokenizer`, `RegexTokenizer`
- tokenizer용 `normalize`
- `NounLMatchTokenizer`, `NounMatchTokenizer`

### `soynlp/tokenizer/_tokenizer.py`

- `RegexTokenizer`: 문자 종류 경계 기반 토크나이저.
- `LTokenizer`: 점수 기반 L+R 분해기.
- `MaxScoreTokenizer`: greedy max-score substring 분해기.
- `MaxLRScoreTokenizer`: L/R 사전, LR graph, 선호도까지 쓰는 실험적 분해기.

### `soynlp/tokenizer/_noun_tokenizer.py`

- `NounLMatchTokenizer`: 왼쪽부터 명사를 최대 길이 매칭.
- `NounMatchTokenizer`: 명사 점수 기반으로 eojeol 안 명사 구간을 찾음.

### `soynlp/tokenizer/_normalizer.py`

- tokenizer 단계에서 쓰는 가벼운 normalize 도구.
- 반복과 emoji 정리를 더 간단한 방식으로 제공.

### `soynlp/tokenizer/_tokenizer_builder.py`

- `EojeolPatternTrainer` 구현.
- substring vocabulary와 LR/RL graph를 훈련하고 저장/로드할 수 있다.

## 5.8 `soynlp.lemmatizer`

### `soynlp/lemmatizer/__init__.py`

- `Lemmatizer`, `lemma_candidate`, `lemma_candidate_chat`
- `conjugate`, `conjugate_chat`, `_conjugate_stem`

### `soynlp/lemmatizer/_conjugation.py`

- stem+ending 조합을 실제 표면형으로 만드는 규칙 엔진.
- 한국어 불규칙 활용 규칙이 가장 밀집해 있는 파일이다.
- `_conjugate_stem()`은 stem 표면형만 생성한다.

### `soynlp/lemmatizer/_lemmatizer.py`

- `Lemmatizer` 클래스 구현.
- `lemma_candidate()`와 `lemma_candidate_chat()`가 표면형에서 기본형 후보를 복원한다.
- predefined 예외 처리와 최종 conjugation 검증 로직이 들어 있다.

## 5.9 `soynlp.noun`

### `soynlp/noun/__init__.py`

- `LRNounExtractor`
- `NewsNounExtractor`
- `LRNounExtractor_v2`
- `NounScore`
- `extract_domain_pos_features`

### `soynlp/noun/_noun_ver1.py`

- 첫 번째 L-R 기반 명사 추출기.
- 단일 `r -> coefficient` 사전을 이용해 noun score를 계산한다.
- `known_r_ratio`를 함께 반환한다.

### `soynlp/noun/_noun_ver2.py`

- 현재 레포의 중심 명사 추출기.
- pos/neg/common feature 집합을 사용한다.
- `LRGraph`, `EojeolCounter`, raw sentences 모두 입력으로 받을 수 있다.
- iterative prediction, compound noun extraction, postprocessing, domain pos feature extraction을 포함한다.

### `soynlp/noun/_noun_news.py`

- 뉴스 도메인 특화 명사 추출기.
- `NewsNounScore` 구조를 사용한다.
- 규칙 기반 후처리가 매우 많고, `r_scores`, `noun_dictionary`, verb/adjective dictionary를 함께 사용한다.

### `soynlp/noun/_josa.py`

- 코퍼스에서 새로운 양의 feature(조사류)를 추가로 뽑는 함수 집합.
- 명사 후보의 왼쪽 다양성과 entropy를 바탕으로 오른쪽 feature를 평가한다.

### `soynlp/noun/_noun_postprocessing.py`

- `detaching_features`
- `ignore_features`
- `check_N_is_NJ`
- 후처리 로그 파일 기록 지원

### `soynlp/noun/frequent_enrolled_josa.txt`

- 후처리에서 사용하는 자주 등장하는 조사 목록.

### `soynlp/noun/frequent_noun_suffix.txt`

- 조사로 오인하면 안 되는 명사 접미 성분 목록.

## 5.10 `soynlp.predicator`

### `soynlp/predicator/__init__.py`

- `EomiExtractor`, `EomiScore`
- `PredicatorExtractor`, `Predicator`
- `StemExtractor`

### `soynlp/predicator/_predicator.py`

- 용언 추출의 메인 총괄 파이프라인.
- 기본 사전 로드, noun-covered eojeol 제거, eomi/stem 추출, eojeol lemmatization, adjective/verb 분리를 담당.

### `soynlp/predicator/_eomi.py`

- 어간 표면형 뒤의 R를 바탕으로 eomi를 추출한다.
- prediction 후 기본형 어미로 복원한다.

### `soynlp/predicator/_stem.py`

- known eomi 뒤에 오는 L 후보로부터 새로운 stem을 추출한다.
- 오른쪽 문자 다양성과 entropy가 중요한 판단 기준이다.

### `soynlp/predicator/_adjective_vs_verb.py`

- conjugation 테스트 기반으로 adjective/verb를 나누는 보조 규칙 모음.
- `rule_classify()`는 suffix에 기반한 빠른 규칙 분류를 제공한다.

## 5.11 `soynlp.pos`

### `soynlp/pos/__init__.py`

- `load_default_adverbs`, `stem_to_adverb`
- `NewsPOSExtractor`, `ChatPOSExtractor`

### `soynlp/pos/_news_pos.py`

- 코퍼스 기반 POS 추출의 핵심 상위 파이프라인.
- noun/predicator를 먼저 추출한 뒤 남은 eojeol에 여러 패턴 매칭을 순차 적용한다.

### `soynlp/pos/_chat_pos.py`

- `NewsPOSExtractor`의 채팅체 변형.
- 잘못된 stem/eomi 제거, noun phrase 제거 같은 규칙이 추가되어 있다.

### `soynlp/pos/_adverb.py`

- 기본 adverb 사전 로드.
- `-하` stem을 `-히` 부사로 바꾸는 간단한 파생 규칙 제공.

## 5.12 `soynlp.postagger`

### `soynlp/postagger/__init__.py`

- 현재 공개되는 것은 `Dictionary`, evaluator들, template matcher들, `SimpleTagger`, postprocessor, `tagset`.
- `LRMaxScoreTagger`는 공개 export가 주석 처리되어 있다.

### `soynlp/postagger/_dictionary.py`

- 가장 단순한 품사 사전 클래스.
- word -> tags 조회, 태그별 단어 추가/삭제, JSON save/load 제공.

### `soynlp/postagger/_template.py`

- `LR` namedtuple 정의.
- `EojeolTemplateMatcher`, `LRTemplateMatcher` 구현.

### `soynlp/postagger/_evaluator.py`

- `BaseEvaluator`
- `SimpleEojeolEvaluator`
- `LREvaluator`
- 후보 분해들을 점수화하고 최적 후보를 선택한다.

### `soynlp/postagger/_tagger.py`

- `BaseTagger`, `SimpleTagger`
- `BasePostprocessor`, `UnknowLRPostprocessor`

### `soynlp/postagger/_lrtagger.py`

- LR graph와 cohesion/droprate를 평가에 넣는 확장 태거 구현.
- 현재 공개 API에서는 빠져 있고, 내부 의존성이 최신 `Dictionary`와 완전히 맞지는 않는 흔적이 있다.

### `soynlp/postagger/_maxscore.py`

- `MaxScoreTagger`라는 매우 작은 스텁 클래스만 있다.

### `soynlp/postagger/_pos_extractor.py`

- noun extractor + predicator extractor를 묶어 POS를 얻으려는 상위 클래스.
- 현재 코드 시그니처를 보면 예전 API를 기준으로 작성된 흔적이 있다.

### `soynlp/postagger/tagset/soy.py`

- 영문 태그명을 한국어 품사명으로 대응시키는 tagset 딕셔너리.

## 5.13 `soynlp.ner`

### `soynlp/ner/__init__.py`

- 날짜/요일/숫자/금액/기간/시간 판별 함수명을 공개한다.

### `soynlp/ner/_rules.py`

- 실제 구현은 아직 없고 모두 `NotImplemented`다.

## 5.14 지금 가장 먼저 봐야 할 파일

파일 단위로 우선순위를 뽑으면 아래와 같다.

1. `soynlp/utils/utils.py`
2. `soynlp/word/_word.py`
3. `soynlp/tokenizer/_tokenizer.py`
4. `soynlp/noun/_noun_ver2.py`
5. `soynlp/lemmatizer/_conjugation.py`
6. `soynlp/lemmatizer/_lemmatizer.py`
7. `soynlp/predicator/_predicator.py`
8. `soynlp/pos/_news_pos.py`

이 여덟 파일을 중심으로 보면 레포의 대부분이 보인다.
