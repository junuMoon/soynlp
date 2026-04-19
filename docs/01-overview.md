# 1. 프로젝트 개요

## 1.1 프로젝트의 정체성

`soynlp`는 한국어 텍스트를 다루기 위한 pure Python 기반 비지도 NLP 툴킷이다. README의 표현을 그대로 풀면, "학습데이터를 이용하지 않으면서 데이터 안에서 반복되는 통계 패턴만으로 단어 경계, 명사, 용언, 품사 단서를 찾는 도구 모음"에 가깝다.

이 프로젝트는 전통적인 형태소 분석기처럼 "정답이 있는 품사 태그 말뭉치"를 전제로 움직이지 않는다. 대신 아래 질문을 반복해서 던진다.

- 같은 글자 조각이 여러 어절에서 얼마나 자주 반복되는가?
- 어떤 글자 조각의 오른쪽에는 주로 조사/어미 같은 기능형 요소가 오는가?
- 어떤 글자 조각은 자체적으로 단어처럼 응집되어 있는가?
- 어떤 표면형은 어떤 기본형 stem/eomi 조합으로 복원될 수 있는가?

즉 이 레포의 중심 사고방식은 "언어학적 정답 표"보다 "코퍼스 안의 분포와 연결 패턴"이다.

## 1.2 레포의 가장 중요한 아이디어

이 코드베이스는 여러 모듈로 나뉘어 있지만, 실제로는 거의 모두 다음 정신모형을 공유한다.

1. 문장을 어절 단위로 본다.
2. 어절 내부 substring을 센다.
3. 어절을 `L + R`로 나눠 생각한다.
4. 왼쪽(L)은 명사/어근/실질형 후보, 오른쪽(R)은 조사/어미/기능형 후보로 해석한다.
5. 자주 등장하고, 다양한 문맥을 가지며, 내부 응집이 높은 조각을 더 "단어답다"고 본다.
6. 한 번 찾아낸 강한 후보는 LR graph에서 제거하여 남은 후보를 더 쉽게 찾는다.

이 마지막 단계가 중요하다. `noun`, `predicator`, `pos` 계열 코드는 대부분 "강한 후보를 먼저 뽑고, 그것이 덮고 있던 어절 패턴을 그래프에서 지운 뒤, 다음 후보를 본다"는 식으로 동작한다.

## 1.3 패키지 단위 지도

아래 표는 현재 `soynlp/` 패키지 내부 코드량과 역할을 함께 요약한 것이다. 줄 수는 현재 소스 트리 기준 대략치다.

| 패키지 | 대략 코드량 | 역할 |
| --- | ---: | --- |
| `noun` | 1600줄 | 명사 후보 추출, 조사(feature) 기반 분류, 복합명사 처리 |
| `predicator` | 1080줄 | 어미/어간 추출, 용언(형용사/동사) 복원 |
| `postagger` | 952줄 | 사전 기반 태거, 템플릿/평가기, 실험적 태거 코드 |
| `tokenizer` | 821줄 | 점수 기반 토크나이저, 정규식 토크나이저, 명사 전용 토크나이저 |
| `lemmatizer` | 666줄 | 활용형 생성, 불규칙 활용 복원, 표면형에서 기본형 후보 추정 |
| `pos` | 647줄 | 코퍼스 기반 품사 추출기, 뉴스/채팅 도메인 특화 흐름 |
| `word` | 599줄 | 단어 점수 계산, PPMI, bigram |
| `utils` | 551줄 | 코퍼스 리더, eojeol counter, LR graph, 메모리/유틸 |
| `hangle` | 310줄 | 한글 음절 분해/조합, 거리 함수, 문자 인코더 |
| `vectorizer` | 296줄 | sparse matrix 벡터화, word-context matrix |
| `normalizer` | 133줄 | 반복 문자/이모티콘/텍스트 정규화 |
| `ner` | 23줄 | 개체명 규칙 함수 자리만 있는 미완성 스텁 |

## 1.4 기능별 큰 흐름

이 레포는 사실상 네 개의 큰 축으로 읽으면 편하다.

### A. 문자/정규화 축

- `hangle`
- `normalizer`

한글 분해/조합, 자모 단위 처리, 반복 이모티콘 정규화를 담당한다. 이 축은 다른 모든 상위 모듈의 바닥이다.

### B. 통계/그래프 축

- `utils`
- `word`
- `vectorizer`

코퍼스를 읽고, 어절과 substring을 세고, L/R 그래프를 만들고, 단어 점수나 word-context 행렬을 계산한다. 이 레포의 "엔진룸"에 해당한다.

### C. 추출/복원 축

- `noun`
- `lemmatizer`
- `predicator`
- `pos`

코퍼스 통계를 언어학적 후보로 바꾸는 핵심 계층이다. 명사, 어미, 어간, 형용사/동사, 최종 품사 집합까지 이 축에서 올라온다.

### D. 분해/태깅 축

- `tokenizer`
- `postagger`

학습된 점수나 사전을 바탕으로 실제 문자열을 잘라내고 태깅한다. `tokenizer`는 점수 기반 분해기이고, `postagger`는 사전 기반 태거다.

## 1.5 가장 중요한 파이프라인

### 파이프라인 1: 단어 점수 기반 토크나이저

`sentences -> WordExtractor.train() -> word scores -> LTokenizer/MaxScoreTokenizer`

이 라인은 "코퍼스에서 의미 있는 substring을 찾고, 그 점수를 토크나이저에 다시 넣는" 구조다.

### 파이프라인 2: 명사 추출

`sentences -> EojeolCounter/LRGraph -> positive/negative R feature 기반 분류 -> noun candidates -> compound noun 후처리`

현재 코드베이스 기준으로는 `LRNounExtractor_v2`가 사실상 주력이다.

### 파이프라인 3: 용언 추출

`nouns -> noun-covered eojeol 제거 -> predicator LRGraph -> eomi 추출 -> stem 추출 -> eojeol lemmatization -> adjective/verb 분리`

이 라인은 `PredicatorExtractor`가 담당한다.

### 파이프라인 4: 코퍼스 기반 POS 추출

`nouns + predicators + adverbs + josas + lemmatization -> eojeol pattern matching -> POS buckets`

이 라인은 `pos/_news_pos.py`, `pos/_chat_pos.py`가 담당한다.

### 파이프라인 5: 사전 기반 POS tagging

`Dictionary -> TemplateMatcher -> Evaluator -> SimpleTagger -> Postprocessor`

이 라인은 `postagger/`가 담당한다. 코퍼스 학습 결과를 직접 쓰기보다, 이미 준비된 품사 사전을 대상으로 태깅한다.

## 1.6 현재 코드 트리 기준 "중심 경로"와 "주변 경로"

소스만 읽어보면 모든 모듈의 성숙도가 같지는 않다. 현재 트리 기준으로는 아래처럼 이해하는 것이 안전하다.

### 중심 경로로 읽어도 좋은 모듈

- `soynlp.hangle`
- `soynlp.normalizer`
- `soynlp.utils`
- `soynlp.word`
- `soynlp.tokenizer`
- `soynlp.lemmatizer`
- `soynlp.noun._noun_ver2`
- `soynlp.predicator`
- `soynlp.pos`
- `soynlp.vectorizer`

이들은 서로의 인터페이스가 비교적 잘 이어지고, README/튜토리얼/테스트에서도 실제 사용 흔적이 많다.

### 역사적/실험적/미완성 흔적이 보이는 모듈

- `soynlp.noun._noun_ver1`
- `soynlp.noun._noun_news`
- `soynlp.postagger._lrtagger`
- `soynlp.postagger._pos_extractor`
- `soynlp.ner._rules`

이 문서는 이 모듈들을 빼지 않고 다루되, "현재 주력 경로"와는 분리해서 설명한다.

## 1.7 레포 바깥쪽 파일들의 의미

소스만 보면 오히려 전체가 잘 안 보인다. 이 레포는 아래 자산들을 함께 봐야 의도가 선명해진다.

- `tutorials/`: 실제 사용법과 설계 배경을 담은 노트북
- `test/`: 간단한 예제 기반 검증 스크립트
- `soynlp/trained_models/`: 명사 추출기가 쓰는 기본 feature 자원
- `soynlp/lemmatizer/dictionary/default/`: 어간/어미 기본 사전
- `soynlp/postagger/dictionary/`: 품사 사전 자산
- `notes/`: 발표자료와 외부 알고리즘 해설

## 1.8 이 레포를 읽을 때 가장 좋은 순서

개인적으로는 아래 순서를 추천한다.

1. `utils`로 코퍼스, eojeol, LR graph를 이해한다.
2. `word`와 `tokenizer`로 substring scoring 감각을 잡는다.
3. `noun/_noun_ver2.py`를 읽어 이 레포의 대표 알고리즘을 본다.
4. `lemmatizer`와 `predicator`로 용언 복원 흐름을 읽는다.
5. `pos/_news_pos.py`로 상위 품사 추출 파이프라인을 본다.
6. 마지막에 `postagger/`를 읽으며 사전 기반 라인과 비교한다.

이 문서 세트도 그 흐름에 맞춰 배치했다.

