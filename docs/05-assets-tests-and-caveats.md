# 5. 자산, 테스트, 그리고 현재 트리 기준 주의사항

이 장은 코드 바깥 자산과 "현재 레포 상태를 읽을 때 놓치기 쉬운 점"을 정리한다.

## 5.1 패키징과 메타데이터

### `setup.py`

- 패키지 이름: `soynlp`
- 버전: 루트 `soynlp.__version__` 사용
- 의존성:
  - `numpy>=1.12.1`
  - `psutil>=5.0.1`
  - `scipy>=1.1.0`
  - `scikit-learn>=0.20.0`
- `README.md`를 long description으로 사용
- 기본 모델/사전 텍스트 파일을 `package_data`에 포함

### 루트 메타정보

- `soynlp/__init__.py`에는 현재 버전 `0.0.493`가 적혀 있다.
- `LICENSE`가 별도로 있으며, `setup.py` classifier는 GPLv3라고 적고 루트 메타데이터는 `LGPL`이라고 적는다. 즉 라이선스 표기는 한 번 더 확인하고 해석하는 편이 안전하다.

## 5.2 학습 자원과 기본 사전

### 명사 추출기 기본 모델

`soynlp/trained_models/` 아래 네 파일이 있다.

- `noun_predictor_sejong`
- `noun_predictor_sejong_logisticregression`
- `noun_predictor_ver2_pos`
- `noun_predictor_ver2_neg`

현재 트리 기준 line 수는 대략 다음과 같다.

- `noun_predictor_ver2_pos`: 4036줄
- `noun_predictor_ver2_neg`: 2428줄
- `noun_predictor_sejong`: 2398줄
- `noun_predictor_sejong_logisticregression`: 2768줄

즉 `LRNounExtractor_v2`는 pos/neg feature 목록 기반, v1/news 계열은 `r -> score` 기반 자원을 쓰는 구조라고 이해하면 된다.

### lemmatizer 기본 사전

`soynlp/lemmatizer/dictionary/default/` 아래에 다음 자원이 있다.

- `Stem/Adjective.txt`
- `Stem/Verb.txt`
- `StemL/Adjective.txt`
- `StemL/Verb.txt`
- `Eomi/Eomi.txt`
- `EomiR/Eomi.txt`

현재 줄 수는 대략 다음과 같다.

- `Stem/Adjective.txt`: 2640줄
- `Stem/Verb.txt`: 6435줄
- `StemL/Adjective.txt`: 4308줄
- `StemL/Verb.txt`: 12404줄
- `Eomi/Eomi.txt`: 10329줄
- `EomiR/Eomi.txt`: 11655줄

즉 predicator 계열은 생각보다 큰 기본 lexical resource 위에서 돌아간다.

### postagger 사전

`soynlp/postagger/dictionary/`는 여러 하위 폴더로 나뉜다.

- `default/`: 기본 태그 사전
- `pos/`: 품사별 word list
- `lr/`: L/R 관련 자산
- `morpheme/`: 형태소 자산
- `sejong/`: 세종 기반 동사/형용사 자산
- `extracted/`: 추출 결과 샘플

주의할 점은, 현재 `default/` 하위의 일부 핵심 파일이 비어 있다는 점이다.

- `default/Adjective/adjective.txt`: 0줄
- `default/Exclamation/exclamation.txt`: 0줄
- `default/Noun/noun.txt`: 0줄
- `default/Numeral/numeral.txt`: 0줄
- `default/Verb/verb.txt`: 0줄

반면 아래 파일들은 실질 내용이 있다.

- `default/Adverb/adverb.txt`: 2551줄
- `default/Josa/josa.txt`: 530줄
- `default/Josa/josa_chat.txt`: 571줄
- `default/Pronoun/pronoun.txt`: 110줄
- `default/Determiner/determiner.txt`: 95줄

즉 postagger 라인은 "샘플/뼈대 + 일부 실질 자원" 성격이 강하다고 보는 편이 자연스럽다.

## 5.3 튜토리얼과 노트

### 튜토리얼 노트북

`tutorials/`는 README보다 훨씬 실용적이다. 각 노트북의 역할은 대략 이렇다.

- `nounextractor-v1_usage.ipynb`: v1 명사 추출기와 OOV 문제 설명
- `nounextractor-v2_usage.ipynb`: v2 명사 추출기 사용법과 복합명사 설명
- `tokenizer_usage.ipynb`: `LTokenizer`, `MaxScoreTokenizer` 계열 설명
- `wordextractor_lecture.ipynb`: unsupervised word extraction 배경과 점수 개념
- `vectorizer_usage.ipynb`: tokenizer와 vectorizer 결합 예제
- `pmi_usage.ipynb`: word-context 행렬과 PPMI 계산
- `normalizer_usage.ipynb`: 반복 문자/이모티콘 정규화 예제
- `tagger_usage.ipynb`: postagger 사용법
- `tagger_lecture.ipynb`: 사전 기반 태거 설계 설명
- `dictionary_building_(noun_and_predicator_extraction).ipynb`: 영화 댓글 데이터로 명사/용언 사전 만들기
- `doublespace_line_corpus_(with_noun_extraction).ipynb`: `DoublespaceLineCorpus`와 noun extraction 설명

### notes

- `notes/unskonlp.pdf`: 발표자료
- `notes/kr_word_rank_explained.md`: KR-WordRank를 자세히 풀어쓴 별도 해설 문서

이 노트들은 `soynlp` 자체 구현만 다루지는 않지만, 저자의 문제의식과 주변 알고리즘 생태계를 이해하는 데 도움을 준다.

## 5.4 테스트

`test/` 폴더는 전형적인 pytest unit test 세트라기보다 "예제 기반 검증 스크립트"에 가깝다.

### 주요 파일

- `basic_test.py`
  - 한글 처리
  - 토크나이저
  - WordExtractor
  - 명사 추출기들
  - postagger
  - PMI
- `lemmatizer_test.py`
  - 여러 활용형을 기본형 후보로 복원하는 예제
- `conjugation_test.py`
  - stem + eomi 조합을 표면형으로 만드는 예제
- `conjugate_root_test.py`
  - `_conjugate_stem()`이 만드는 표면 stem 후보 확인
- `lemmatizer_test.ipynb`, `tagger_test.ipynb`
  - notebook 기반 실험
- `basic_test.log`
  - 과거 실행 로그

즉 "자동 검증 체계"라기보다 "구현 의도와 예제 출력"을 보여주는 보조 자료다.

## 5.5 예제 데이터

### `tutorials/` 데이터

- `doublespace_line_corpus_sample.txt`
- `2016-10-20.txt`
- `figs/*.JPG`

README와 노트북에서 직접 인용하는 샘플이다.

### `data/` 폴더

현재 여섯 개의 텍스트가 있다.

- `91031.txt`, `91031_norm.txt`
- `99714.txt`, `99714_norm.txt`
- `134963.txt`, `134963_norm.txt`

줄 수를 보면 정규화 전/후 파일 쌍으로 보인다.

- 91031 계열: 19902줄
- 99714 계열: 13814줄
- 134963 계열: 15603줄

코드에서 직접 참조되지는 않지만, 정규화나 외부 실험 산출물로 보기에 자연스럽다.

## 5.6 GitHub Actions

`.github/workflows/pytest.yaml`이 존재한다.

의도 자체는 명확하다.

- PR 시 Ubuntu + Python 3.x에서 테스트
- `pytest`, `pytest-cov`, `requirements.txt` 설치
- `tests/*`에 대해 pytest 실행

다만 현재 트리 기준으로 보면 아래와 같은 차이가 있다.

- `requirements.txt`가 없다.
- 실제 테스트 폴더는 `tests/`가 아니라 `test/`다.

즉 workflow는 현재 소스 트리와 완전히 동기화되어 있지는 않은 것으로 보인다.

## 5.7 현재 트리 기준 주의사항

아래 항목들은 "코드를 이해할 때 알고 있으면 혼동을 줄이는 점"이다. 여기서는 동작 여부를 단정하기보다, 소스 상의 불일치나 미완성 흔적을 정리한다.

### 1. `NewsNounExtractor`는 오래된 흔적이 보인다

`soynlp/noun/_noun_news.py`의 constructor는 `base_noun_dictionary` 인자를 받지만, 내부에서는 `noun_dictionary`라는 이름을 사용한다. 소스만 보면 현재형 리팩터링이 덜 끝난 흔적처럼 읽힌다.

### 2. `postagger/_lrtagger.py`는 현재 `Dictionary` 구현과 결이 다르다

이 파일은 `Dictionary(domain_dictionary_folders, use_base_dictionary, ...)` 같은 richer API와 `pos_L`, `pos_R`, `save_domain_dictionary()` 메서드를 기대한다. 그러나 현재 공개된 `postagger/_dictionary.py`의 `Dictionary`는 훨씬 단순하다. 그래서 `_lrtagger.py`는 과거/실험적 라인으로 읽는 편이 맞다.

### 3. `postagger/_pos_extractor.py`는 예전 extractor 시그니처를 기준으로 작성된 것으로 보인다

현재 `LRNounExtractor_v2`, `PredicatorExtractor`의 인자 이름과 이 파일에서 사용하는 인자 이름이 완전히 같지 않다. 따라서 설계 아이디어 참고용으로는 좋지만, 현재 중심 API와 1:1로 맞물리는 최신 경로라고 보기는 어렵다.

### 4. `ner/_rules.py`는 구현되지 않았다

NER 관련 함수 이름은 export되지만 실제 내용은 모두 `NotImplemented`다. 따라서 이 패키지는 설계 자리만 있는 상태로 보는 편이 정확하다.

### 5. `ensure_normalized` 의미가 모듈마다 완전히 일관적이지 않다

`noun/_noun_ver1.py`, `noun/_noun_ver2.py`, `predicator/_predicator.py`를 읽어 보면, `ensure_normalized` 플래그가 실제로 normalize 함수를 적용하는 방식이 완전히 동일하지 않다. 문서를 읽고 코드를 사용할 때 이 이름을 "항상 같은 뜻"으로 받아들이면 혼동될 수 있다.

### 6. 일부 구현은 "현재 주력 API"보다 "연구 기록"에 가깝다

대표적으로 아래 파일들은 README 중심 동선보다, 연구 과정의 분기나 미완성 확장선처럼 보인다.

- `soynlp/noun/_noun_news.py`
- `soynlp/postagger/_lrtagger.py`
- `soynlp/postagger/_maxscore.py`
- `soynlp/postagger/_pos_extractor.py`

이 문서 세트는 이 파일들을 숨기지 않지만, 중심선과 분리해 읽도록 안내한 이유가 여기 있다.

## 5.8 이 장에서 기억할 것

- 이 레포는 코드만이 아니라 모델 파일, 기본 사전, 튜토리얼, 테스트 스크립트가 함께 의미를 만든다.
- `noun_v2`, `predicator`, `pos`, `tokenizer`, `word`가 현재 이해의 중심선이다.
- 일부 모듈과 CI 설정은 현재 트리와 완전히 맞지는 않는 역사적 흔적을 보인다.

