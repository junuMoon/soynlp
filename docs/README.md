# soynlp 레포 읽기 가이드

이 `docs/` 폴더는 "코드를 다시 하나씩 열어보지 않아도 이 레포의 구조, 핵심 아이디어, 각 모듈의 역할, 남아 있는 실험적 흔적까지 이해할 수 있게" 만드는 것을 목표로 작성했다.

이 문서들은 소스 트리의 실제 파일을 기준으로 정리했다. 특히 `soynlp/` 패키지, `tutorials/`, `notes/`, `test/`, `trained_models/`, 기본 사전 파일을 함께 읽는 관점으로 묶어두었다.

## 추천 읽기 순서

이제 문서 세트는 `본편`과 `부록`으로 나뉜다.

### 본편: 처음부터 끝까지 읽는 흐름

1. [01-overview.md](01-overview.md)
2. [02-core-abstractions.md](02-core-abstractions.md)
3. [03-algorithm-walkthrough.md](03-algorithm-walkthrough.md)
4. [04-implementation-deep-dive.md](04-implementation-deep-dive.md)

### 부록: 읽다가 막힐 때 찾아보는 문서

5. [05-module-reference.md](05-module-reference.md)
6. [06-assets-tests-and-caveats.md](06-assets-tests-and-caveats.md)
7. [07-file-index.md](07-file-index.md)

### 보충 읽기

8. [kr_word_rank_explained.md](kr_word_rank_explained.md)

### 설계 제안

9. [08-rearchitecture-proposal.md](08-rearchitecture-proposal.md)

## 이 레포를 한 문장으로 요약하면

`soynlp`는 "주석 말뭉치 없이 한국어 코퍼스에서 반복 패턴을 세고, 어절을 L(왼쪽 의미 단위) + R(오른쪽 기능 단위)로 바라보며, 단어/명사/용언/품사를 통계적으로 추정하는 비지도 한국어 NLP 툴킷"이다.

## 먼저 알고 가면 좋은 결론

- 이 레포의 중심은 `어절 빈도 -> substring/LR graph -> 점수화 -> 추출/분해` 흐름이다.
- 가장 중요한 패키지는 `utils`, `word`, `tokenizer`, `noun`, `predicator`, `pos`다.
- `lemmatizer`는 여러 추출기의 기반 규칙 엔진이다.
- `postagger`는 "사전 기반 품사 태거" 라인이고, `pos`는 "코퍼스 기반 품사 추출기" 라인이다.
- 레포 안에는 현재 공개 경로와 잘 맞는 코드도 있고, 오래된 실험 코드나 API가 바뀌며 남은 흔적도 함께 있다.

## 이 문서 세트의 범위

- 포함:
  - `soynlp/` 아래의 소스 코드
  - 학습 자원과 기본 사전
  - 튜토리얼 노트북과 예제 데이터
  - 테스트 스크립트
  - 패키징 파일과 GitHub Actions 설정
- 의도적으로 읽기 대상에서 제외:
  - `.git/`, `.jj/`
  - `.venv/`
  - `build/`
  - `soynlp.egg-info/`
  - `__pycache__/`
  - Finder/OS 산출물 (`.DS_Store` 등)

## 어떤 문서가 무엇을 담당하나

### 본편

- [01-overview.md](01-overview.md): 이 프로젝트가 무엇을 만들고 어떤 철학으로 구성됐는지 설명한다.
- [02-core-abstractions.md](02-core-abstractions.md): 거의 모든 모듈이 공유하는 핵심 자료구조와 점수 개념을 설명한다.
- [03-algorithm-walkthrough.md](03-algorithm-walkthrough.md): 실제 추출기와 토크나이저가 어떻게 동작하는지 파이프라인 순서대로 풀어쓴다.
- [04-implementation-deep-dive.md](04-implementation-deep-dive.md): 실제 클래스, 메서드, 내부 상태 변화까지 따라가는 구현 중심 문서다.

### 부록

- [05-module-reference.md](05-module-reference.md): 소스 파일별 역할과 공개 API를 파일 단위로 정리한다.
- [06-assets-tests-and-caveats.md](06-assets-tests-and-caveats.md): 사전, 모델, 튜토리얼, 테스트, 현재 코드 트리 기준 주의점을 정리한다.
- [07-file-index.md](07-file-index.md): 레포의 실제 파일을 빠짐없이 훑을 수 있는 인덱스다.

### 보충

- [kr_word_rank_explained.md](kr_word_rank_explained.md): 이 레포와 문제의식이 닿아 있는 KR-WordRank를 별도 배경 지식으로 자세히 풀어쓴 보충 문서다.

### 설계 제안

- [08-rearchitecture-proposal.md](08-rearchitecture-proposal.md): WordRank, KR-WordRank, 그리고 현재 soynlp 구현을 하나의 공통 플랫폼으로 다시 세운다면 어떤 구조가 좋은지 제안한다.

## 빠른 길찾기

- "이 프로젝트를 처음부터 순서대로 이해하고 싶다"면 `01 -> 02 -> 03 -> 04`만 먼저 읽으면 된다.
- "특정 클래스나 파일이 무슨 역할인지 빨리 찾고 싶다"면 `05`와 `07`부터 보면 된다.
- "현재 기준으로 어디가 주력 경로이고 어디가 실험적 흔적인지" 알고 싶다면 `06`을 먼저 보면 된다.
- "KR-WordRank까지 함께 묶어서 이해하고 싶다"면 마지막에 보충 문서를 덧붙이면 된다.
- "이 레포를 완전히 다시 짠다면 어떤 구조가 맞을지" 보고 싶다면 설계 제안 문서를 보면 된다.
