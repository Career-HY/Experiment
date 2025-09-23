# Career-HY RAG 실험 파이프라인

Career-HY RAG 시스템의 다양한 파라미터를 체계적으로 실험하여 검색 성능을 최적화하기 위한 파이프라인입니다.

## 🎯 프로젝트 개요

### 목적
- **검색 성능 최적화**: 다양한 임베딩 모델, 청킹 전략, 검색 알고리즘 비교
- **체계적 실험**: YAML 기반 설정으로 재현 가능한 실험 환경 제공
- **비용 효율성**: 임베딩 캐싱을 통한 OpenAI API 비용 절약

<br>

## 📁 디렉토리 구조

```
Experiment/
├── configs/                 # 실험 설정 파일들
│   ├── baseline.yaml       # 베이스라인 설정 (현재 서비스)
│   └── chunking_test.yaml  # 청킹 전략 실험 설정
├── core/                   # 핵심 파이프라인
│   ├── interfaces/        # 추상 인터페이스 (ABC)
│   ├── pipeline.py        # 메인 실험 파이프라인
│   └── config.py         # 설정 관리
├── implementations/       # 구현체들
│   ├── embedders/        # 임베딩 모델 (OpenAI 등)
│   ├── chunkers/         # 청킹 전략 (RecursiveCharacterTextSplitter 등)
│   ├── retrievers/       # 검색 시스템 (ChromaDB 등)
│   └── evaluators/       # 평가 지표 계산
├── utils/                # 유틸리티
│   ├── data_loader.py    # S3 데이터 로드
│   ├── embedding_cache.py # 임베딩 캐싱 시스템
│   ├── gt_converter.py   # Ground Truth CSV→JSONL 변환기
│   └── factory.py        # 컴포넌트 팩토리
├── data/                 # Ground Truth 데이터
│   ├── test_queries.jsonl      # 테스트 쿼리 (575개)
│   ├── test_queries_small.jsonl # 작은 테스트 셋 (3개)
│   └── ground_truth.jsonl      # Ground Truth 데이터
├── cache/                # 임베딩 캐시 저장소
├── results/              # 실험 결과
├── run_experiment.sh     # 메인 실험 실행 스크립트
├── docker-compose.yml    # Docker 구성
├── Dockerfile           # Docker 이미지 정의
└── requirements.txt     # Python 의존성
```

<br>

## 실험 진행 가이드
[실험 진행 가이드 바로가기](EXPERIMENT_GUIDE.md)

<br>

## 🔧 지원하는 구현체들 (실험 진행하면서 추가해갈 예정)

### 임베딩 모델
- **OpenAI**: `text-embedding-ada-002`

### 청킹 전략
- **no_chunk**: 청킹 없음 (전체 문서 사용)
- **recursive**: RecursiveCharacterTextSplitter (설정 가능한 크기/오버랩)

### 검색 시스템
- **ChromaDB**: 벡터 데이터베이스 (코사인 유사도)

<br>

## 평가 지표

#### Recall@k (재현율)
- **정의**: 전체 관련 문서 중 상위 k개 검색 결과에 포함된 관련 문서의 비율
- **계산**: `Recall@k = (상위 k개 중 관련 문서 수) / (전체 관련 문서 수)`
- **의미**: 놓친 관련 문서가 얼마나 적은지 측정 (높을수록 좋음)
- **예시**: 관련 문서 10개 중 상위 5개에서 3개 발견 → Recall@5 = 0.3

#### Precision@k (정밀도)
- **정의**: 상위 k개 검색 결과 중 실제로 관련 있는 문서의 비율
- **계산**: `Precision@k = (상위 k개 중 관련 문서 수) / k`
- **의미**: 검색 결과의 정확성 측정 (높을수록 좋음)
- **예시**: 상위 5개 중 3개가 관련 있음 → Precision@5 = 0.6

#### MRR (Mean Reciprocal Rank)
- **정의**: 각 쿼리의 첫 번째 관련 문서 순위의 역수 평균
- **계산**: `MRR = (1/|Q|) × Σ(1/rank_i)` (rank_i = 첫 번째 관련 문서 순위)
- **의미**: 관련 문서가 얼마나 상위에 위치하는지 측정 (높을수록 좋음)
- **예시**: 첫 관련 문서가 3번째 → RR = 1/3 = 0.333

#### MAP (Mean Average Precision)
- **정의**: 각 쿼리의 Average Precision의 평균값
- **계산**: `MAP = (1/|Q|) × Σ(AP_i)` (AP = 관련 문서별 Precision 평균)
- **의미**: 모든 관련 문서 순위를 고려한 종합적 성능 (높을수록 좋음)
- **특징**: Precision과 Recall을 모두 반영한 균형 잡힌 지표

#### nDCG@k (Normalized Discounted Cumulative Gain)
- **정의**: 상위 k개 결과의 순위별 가중 점수를 이상적 순위와 비교한 정규화 점수
- **계산**: `nDCG@k = DCG@k / IDCG@k`
- **의미**: 순위가 높을수록 더 중요하다고 가정한 성능 측정 (높을수록 좋음)
- **특징**: 상위 순위에 있는 관련 문서에 더 높은 가중치 부여

<br>

## 💾 임베딩 캐싱 시스템

### 캐시 동작 원리
1. **캐시 키 생성**: `{embedding_model}_{chunking_strategy}`
2. **첫 실행**: OpenAI API 호출 → 임베딩 생성 → 캐시 저장
3. **재실행**: 캐시 확인 → 기존 임베딩 로드 (API 호출 없음)

### 캐시 파일 구조
```
cache/embeddings/{cache_key}/
├── embeddings.npy          # NumPy 배열 (1536차원 벡터들)
├── processed_documents.pkl # 처리된 문서 텍스트
└── metadata.json          # 캐시 메타데이터
```

### 캐시 관리

```bash
# 캐시 목록 확인
ls cache/embeddings/

# 특정 캐시 삭제 (새로운 임베딩 생성하려면)
rm -rf cache/embeddings/text_embedding_ada_002_no_chunk

# 전체 캐시 삭제
rm -rf cache/embeddings/*
```

## 📈 실험 결과 분석

### 결과 파일들
실험 완료 후 `results/{experiment_name}/` 디렉토리에 저장:

```
results/baseline/
├── results_{experiment_id}.json      # 주요 지표 요약
└── detailed_results_{experiment_id}.jsonl # 상세 쿼리별 결과
```


## 📊 데이터 소스

### S3 데이터
- **버킷**: `career-hi`
- **PDF 경로**: `initial-dataset/pdf/` (1,473개 파일)
- **JSON 경로**: `initial-dataset/json/` (1,473개 파일)
- **페이지네이션**: 모든 S3 객체 자동 처리

### Ground Truth
- **테스트 쿼리**: 575개 (Agent를 활용해 생성 / 실제 사용자 프로필과 유사하도록)
- **Ground Truth**: 각 쿼리별 관련 채용공고 ID 매핑

<br>

## 🔄 Ground Truth 데이터 관리

### 새로운 버전의 GT CSV → JSONL 변환

새로운 Ground Truth CSV 파일을 받았을 때 실험에 사용할 JSONL 형태로 변환하는 유틸리티:

```bash
python utils/gt_converter.py new_ground_truth.csv data/test_queries_new.jsonl
```


<br>


