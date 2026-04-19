# 7. 파일 인덱스

이 문서는 현재 레포에서 "읽을 가치가 있는 핵심 파일"을 빠짐없이 훑기 위한 인덱스다.

제외한 항목:

- `.git/`, `.jj/`
- `.venv/`
- `build/`
- `soynlp.egg-info/`
- `__pycache__/`
- OS 산출물과 임시 환경 파일

## 7.1 루트 파일

- `.gitattributes`: Git 속성 설정.
- `.gitignore`: 무시 파일 목록.
- `.github/workflows/pytest.yaml`: CI 설정. 현재 트리와 일부 불일치가 있다.
- `LICENSE`: 라이선스 파일.
- `README.md`: 프로젝트 공식 소개와 사용 예제.
- `setup.py`: 패키징 설정.

## 7.1a `docs/` 폴더

- `docs/README.md`: 문서 세트의 시작점과 읽기 순서.
- `docs/01-overview.md`: 본편 1권. 프로젝트 정체성과 아키텍처 요약.
- `docs/02-core-abstractions.md`: 본편 2권. 공통 자료구조와 용어 정리.
- `docs/03-algorithm-walkthrough.md`: 본편 3권. 알고리즘 파이프라인 설명.
- `docs/04-implementation-deep-dive.md`: 본편 4권. 구현 중심 상세 해설.
- `docs/05-module-reference.md`: 부록 A. 소스 파일 레퍼런스.
- `docs/06-assets-tests-and-caveats.md`: 부록 B. 자산/테스트/주의사항.
- `docs/07-file-index.md`: 부록 C. 전체 파일 인덱스.
- `docs/kr_word_rank_explained.md`: KR-WordRank 보충 해설 문서.
- `docs/08-rearchitecture-proposal.md`: 미래 구조 재구성 최종안.

## 7.2 `soynlp/` 루트

- `soynlp/__init__.py`: 패키지 메타데이터와 공개 모듈 export.

## 7.3 `soynlp/hangle/`

- `soynlp/hangle/__init__.py`: 문자 처리 유틸 export.
- `soynlp/hangle/_hangle.py`: 한글 분해/조합, 문자 판별, encoder.
- `soynlp/hangle/_distance.py`: 문자열 거리 함수.

## 7.4 `soynlp/normalizer/`

- `soynlp/normalizer/__init__.py`: normalize 관련 공개 함수 export.
- `soynlp/normalizer/_normalizer.py`: 반복 문자, 이모티콘, 문자 필터 정규화 구현.

## 7.5 `soynlp/utils/`

- `soynlp/utils/__init__.py`: core utility export.
- `soynlp/utils/utils.py`: corpus reader, counter, LR graph, misc 유틸.
- `soynlp/utils/math.py`: randomized SVD wrapper.

## 7.6 `soynlp/word/`

- `soynlp/word/__init__.py`: word 관련 공개 API export.
- `soynlp/word/_word.py`: `WordExtractor` 구현.
- `soynlp/word/_pmi.py`: PMI/PPMI 계산.
- `soynlp/word/_phrase.py`: `Bigram` 추출기.

## 7.7 `soynlp/vectorizer/`

- `soynlp/vectorizer/__init__.py`: vectorizer export.
- `soynlp/vectorizer/_vectorizer.py`: sparse vectorizer 구현.
- `soynlp/vectorizer/_word_context.py`: word-context matrix 생성.

## 7.8 `soynlp/tokenizer/`

- `soynlp/tokenizer/__init__.py`: tokenizer export.
- `soynlp/tokenizer/_tokenizer.py`: `RegexTokenizer`, `LTokenizer`, `MaxScoreTokenizer`, `MaxLRScoreTokenizer`.
- `soynlp/tokenizer/_noun_tokenizer.py`: 명사 전용 토크나이저들.
- `soynlp/tokenizer/_normalizer.py`: tokenizer용 가벼운 normalize.
- `soynlp/tokenizer/_tokenizer_builder.py`: `EojeolPatternTrainer`.

## 7.9 `soynlp/lemmatizer/`

- `soynlp/lemmatizer/__init__.py`: lemmatizer 공개 API export.
- `soynlp/lemmatizer/_conjugation.py`: 활용형 생성 규칙.
- `soynlp/lemmatizer/_lemmatizer.py`: 기본형 후보 복원.
- `soynlp/lemmatizer/tag/soy.json`: 태그 관련 자산.

### 기본 사전

- `soynlp/lemmatizer/dictionary/default/Eomi/Eomi.txt`: 기본 어미 사전.
- `soynlp/lemmatizer/dictionary/default/EomiR/Eomi.txt`: 표면형 어미 자원.
- `soynlp/lemmatizer/dictionary/default/Stem/Adjective.txt`: 형용사 어간 사전.
- `soynlp/lemmatizer/dictionary/default/Stem/Verb.txt`: 동사 어간 사전.
- `soynlp/lemmatizer/dictionary/default/StemL/Adjective.txt`: 형용사 어간 표면형 자원.
- `soynlp/lemmatizer/dictionary/default/StemL/Verb.txt`: 동사 어간 표면형 자원.

## 7.10 `soynlp/noun/`

- `soynlp/noun/__init__.py`: noun 관련 공개 API export.
- `soynlp/noun/_noun_ver1.py`: 초기 L-R 기반 명사 추출기.
- `soynlp/noun/_noun_ver2.py`: 현재 중심 명사 추출기.
- `soynlp/noun/_noun_news.py`: 뉴스 도메인 특화 명사 추출기.
- `soynlp/noun/_josa.py`: 도메인 양의 feature 추출 로직.
- `soynlp/noun/_noun_postprocessing.py`: 후처리 함수들.
- `soynlp/noun/frequent_enrolled_josa.txt`: 조사 목록 자원.
- `soynlp/noun/frequent_noun_suffix.txt`: 명사 suffix 목록 자원.

## 7.11 `soynlp/predicator/`

- `soynlp/predicator/__init__.py`: predicator 공개 API export.
- `soynlp/predicator/_predicator.py`: 용언 추출 orchestrator.
- `soynlp/predicator/_eomi.py`: 어미 추출기.
- `soynlp/predicator/_stem.py`: 어간 추출기.
- `soynlp/predicator/_adjective_vs_verb.py`: 형용사/동사 분리 보조 규칙.

## 7.12 `soynlp/pos/`

- `soynlp/pos/__init__.py`: POS extractor export.
- `soynlp/pos/_news_pos.py`: 뉴스 텍스트용 POS 추출기.
- `soynlp/pos/_chat_pos.py`: 채팅체용 POS 추출기.
- `soynlp/pos/_adverb.py`: 부사 사전과 stem->adverb 변환.

## 7.13 `soynlp/postagger/`

- `soynlp/postagger/__init__.py`: 사전 기반 태거 관련 공개 API export.
- `soynlp/postagger/_dictionary.py`: 품사 사전 래퍼.
- `soynlp/postagger/_template.py`: 템플릿 matcher와 `LR` namedtuple.
- `soynlp/postagger/_evaluator.py`: 후보 평가기.
- `soynlp/postagger/_tagger.py`: `SimpleTagger`와 postprocessor.
- `soynlp/postagger/_lrtagger.py`: LR graph 기반 확장 태거 실험선.
- `soynlp/postagger/_maxscore.py`: 최소 구현 스텁.
- `soynlp/postagger/_pos_extractor.py`: 예전 상위 POS 추출기.
- `soynlp/postagger/tagset/__init__.py`: tagset 패키지 init.
- `soynlp/postagger/tagset/soy.py`: 태그명 매핑.

### 기본/보조 사전 자산

- `soynlp/postagger/dictionary/default/README.md`: 기본 사전 포맷 설명.
- `soynlp/postagger/dictionary/default/Adjective/adjective.txt`: 기본 형용사 사전.
- `soynlp/postagger/dictionary/default/Adverb/adverb.txt`: 기본 부사 사전.
- `soynlp/postagger/dictionary/default/Determiner/determiner.txt`: 기본 관형사 사전.
- `soynlp/postagger/dictionary/default/Exclamation/exclamation.txt`: 기본 감탄사 사전.
- `soynlp/postagger/dictionary/default/Josa/josa.txt`: 기본 조사 사전.
- `soynlp/postagger/dictionary/default/Josa/josa_chat.txt`: 채팅용 조사 사전.
- `soynlp/postagger/dictionary/default/Noun/noun.txt`: 기본 명사 사전.
- `soynlp/postagger/dictionary/default/Numeral/numeral.txt`: 기본 수사 사전.
- `soynlp/postagger/dictionary/default/Pronoun/pronoun.txt`: 기본 대명사 사전.
- `soynlp/postagger/dictionary/default/Verb/verb.txt`: 기본 동사 사전.
- `soynlp/postagger/dictionary/extracted/Noun_.txt`: 추출된 명사 예시 자산.
- `soynlp/postagger/dictionary/extracted/scoretable_.csv`: 추출 점수 예시 CSV.
- `soynlp/postagger/dictionary/lr/Adjective.txt`: LR용 형용사 자산.
- `soynlp/postagger/dictionary/lr/Verb.txt`: LR용 동사 자산.
- `soynlp/postagger/dictionary/morpheme/AdjectiveRoot.txt`: 형용사 어근 자산.
- `soynlp/postagger/dictionary/morpheme/Eomi.txt`: 어미 자산.
- `soynlp/postagger/dictionary/morpheme/VerbRoot.txt`: 동사 어근 자산.
- `soynlp/postagger/dictionary/pos/Adjective/Adjective.txt`: 품사별 형용사 목록.
- `soynlp/postagger/dictionary/pos/Adverb/Adverb.txt`: 품사별 부사 목록.
- `soynlp/postagger/dictionary/pos/Determiner/Determiner.txt`: 품사별 관형사 목록.
- `soynlp/postagger/dictionary/pos/Exclamation/Exclamation.txt`: 품사별 감탄사 목록.
- `soynlp/postagger/dictionary/pos/Josa/Josa.txt`: 품사별 조사 목록.
- `soynlp/postagger/dictionary/pos/Noun/Noun.txt`: 품사별 명사 목록.
- `soynlp/postagger/dictionary/pos/Verb/Verb.txt`: 품사별 동사 목록.
- `soynlp/postagger/dictionary/sejong/Adjective.txt`: 세종 기반 형용사 자원.
- `soynlp/postagger/dictionary/sejong/Verb.txt`: 세종 기반 동사 자원.

## 7.14 `soynlp/ner/`

- `soynlp/ner/__init__.py`: NER 규칙 함수 export.
- `soynlp/ner/_rules.py`: 아직 미구현 스텁.

## 7.15 `soynlp/trained_models/`

- `soynlp/trained_models/noun_predictor_sejong`: v1/news 계열 명사 feature 점수 자원.
- `soynlp/trained_models/noun_predictor_sejong_logisticregression`: 세종 기반 logistic regression 자원.
- `soynlp/trained_models/noun_predictor_ver2_pos`: v2 명사 추출용 양성 feature 목록.
- `soynlp/trained_models/noun_predictor_ver2_neg`: v2 명사 추출용 음성 feature 목록.

## 7.16 `test/`

- `test/basic_test.py`: 한글 처리, tokenizer, noun, postagger, PMI 등 종합 예제 테스트.
- `test/basic_test.log`: 과거 실행 로그.
- `test/lemmatizer_test.py`: lemmatizer 예제 실행 스크립트.
- `test/conjugation_test.py`: conjugation 예제 실행 스크립트.
- `test/conjugate_root_test.py`: `_conjugate_stem()` 예제 실행 스크립트.
- `test/lemmatizer_test.ipynb`: lemmatizer notebook 실험.
- `test/tagger_test.ipynb`: tagger notebook 실험.

## 7.17 `tutorials/`

- `tutorials/2016-10-20.txt`: 튜토리얼용 텍스트 예제.
- `tutorials/doublespace_line_corpus_sample.txt`: `DoublespaceLineCorpus` 샘플.
- `tutorials/dictionary_building_(noun_and_predicator_extraction).ipynb`: 명사/용언 사전 구축 노트북.
- `tutorials/doublespace_line_corpus_(with_noun_extraction).ipynb`: corpus 포맷과 noun extraction 노트북.
- `tutorials/normalizer_usage.ipynb`: normalizer 사용법.
- `tutorials/nounextractor-v1_usage.ipynb`: v1 명사 추출기 사용법.
- `tutorials/nounextractor-v2_usage.ipynb`: v2 명사 추출기 사용법.
- `tutorials/pmi_usage.ipynb`: PMI 사용법.
- `tutorials/tagger_lecture.ipynb`: 태거 설계 노트.
- `tutorials/tagger_usage.ipynb`: 태거 사용법.
- `tutorials/tokenizer_usage.ipynb`: tokenizer 사용법.
- `tutorials/vectorizer_usage.ipynb`: vectorizer 사용법.
- `tutorials/wordextractor_lecture.ipynb`: 단어 추출 강의 노트.

### 튜토리얼 그림

- `tutorials/figs/branching_entropy.JPG`: branching entropy 설명 그림.
- `tutorials/figs/bothside_branching_entropy.JPG`: 양방향 branching entropy 설명 그림.
- `tutorials/figs/cohesion_score.JPG`: cohesion 설명 그림.

## 7.18 `notes/`

- `notes/unskonlp.pdf`: 발표 자료.
- `notes/kr_word_rank_explained.md`: KR-WordRank 상세 해설.

## 7.19 `data/`

- `data/91031.txt`: 실험용 텍스트.
- `data/91031_norm.txt`: 정규화된 텍스트 추정 파일.
- `data/99714.txt`: 실험용 텍스트.
- `data/99714_norm.txt`: 정규화된 텍스트 추정 파일.
- `data/134963.txt`: 실험용 텍스트.
- `data/134963_norm.txt`: 정규화된 텍스트 추정 파일.

## 7.20 읽기 우선순위가 높은 파일 묶음

파일 인덱스를 다 보되, 시간이 없으면 아래 묶음부터 보면 된다.

### 핵심 설계 이해용

- `README.md`
- `soynlp/utils/utils.py`
- `soynlp/word/_word.py`
- `soynlp/tokenizer/_tokenizer.py`
- `soynlp/noun/_noun_ver2.py`
- `soynlp/lemmatizer/_conjugation.py`
- `soynlp/lemmatizer/_lemmatizer.py`
- `soynlp/predicator/_predicator.py`
- `soynlp/pos/_news_pos.py`

### 사용 예제 이해용

- `test/basic_test.py`
- `tutorials/nounextractor-v2_usage.ipynb`
- `tutorials/tokenizer_usage.ipynb`
- `tutorials/vectorizer_usage.ipynb`
- `tutorials/pmi_usage.ipynb`

### 역사와 주변 실험 이해용

- `soynlp/noun/_noun_ver1.py`
- `soynlp/noun/_noun_news.py`
- `soynlp/postagger/_lrtagger.py`
- `soynlp/postagger/_pos_extractor.py`
- `notes/kr_word_rank_explained.md`
