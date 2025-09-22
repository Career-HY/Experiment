# Career-HY RAG 실험 파이프라인

Career-HY RAG 시스템의 다양한 파라미터를 체계적으로 실험하여 검색 성능을 최적화하기 위한 파이프라인입니다.

## 🎯 프로젝트 개요

### 목적
- **검색 성능 최적화**: 다양한 임베딩 모델, 청킹 전략, 검색 알고리즘 비교
- **체계적 실험**: YAML 기반 설정으로 재현 가능한 실험 환경 제공
- **비용 효율성**: 임베딩 캐싱을 통한 OpenAI API 비용 절약

### 주요 특징
- 🐳 **Docker 기반**: 일관된 실험 환경 보장
- 💾 **임베딩 캐싱**: 동일 설정 재실험 시 API 비용 절약
- 📊 **다양한 평가 지표**: Recall@k, Precision@k, MRR, MAP, nDCG@k
- 🔧 **모듈형 아키텍처**: 쉬운 확장성과 유지보수
- 📝 **YAML 설정**: 코드 수정 없이 실험 파라미터 조정

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

## 🚀 빠른 시작

### 1. 환경 설정

```bash
# 1. .env 파일 생성 (API 키 설정)
cat > .env << 'EOF'
# OpenAI API 설정
OPENAI_API_KEY=your_openai_api_key_here

# AWS 설정 (S3 접근용)
AWS_ACCESS_KEY_ID=your_aws_access_key_here
AWS_SECRET_ACCESS_KEY=your_aws_secret_key_here
AWS_DEFAULT_REGION=ap-northeast-2

# S3 버킷 설정
S3_BUCKET_NAME=career-hi
EOF

# 2. Docker 확인
docker --version
docker compose --version
```

### 2. 베이스라인 실험 실행

```bash
# 베이스라인 실험 (현재 서비스와 동일한 설정)
./run_experiment.sh configs/baseline.yaml
```

### 3. 청킹 전략 실험

```bash
# 청킹 전략 실험 (RecursiveCharacterTextSplitter)
./run_experiment.sh configs/chunking_test.yaml
```

## 📊 실험 설정

### YAML 설정 파일 구조

```yaml
# 실험 기본 정보
experiment_name: "baseline"
description: "현재 서비스와 동일한 베이스라인 설정"
output_dir: "results"

# 임베딩 설정
embedder:
  type: "openai"
  model_name: "text-embedding-ada-002"
  batch_size: 5

# 청킹 설정
chunker:
  type: "no_chunk"  # 또는 "recursive"
  chunk_size: null
  chunk_overlap: null

# 검색 시스템 설정
retriever:
  type: "chroma"
  collection_name: "job-postings-baseline"
  persist_directory: "/tmp/chroma_baseline"
  top_k: 10

# LLM 설정
llm:
  type: "openai"
  model_name: "gpt-4o-mini"
  temperature: 0.7
  max_tokens: 1000

# 데이터 설정
data:
  s3_bucket: "career-hi"
  pdf_prefix: "initial-dataset/pdf/"
  json_prefix: "initial-dataset/json/"
  test_queries_path: "data/test_queries.jsonl"

# 평가 설정
evaluation:
  metrics: ["recall@k", "precision@k", "mrr", "map", "ndcg@k"]
  k_values: [1, 3, 5, 10]
```

## 🔧 지원하는 구현체들

### 임베딩 모델
- **OpenAI**: `text-embedding-ada-002`, `text-embedding-3-small`, `text-embedding-3-large`

### 청킹 전략
- **no_chunk**: 청킹 없음 (전체 문서 사용)
- **recursive**: RecursiveCharacterTextSplitter (설정 가능한 크기/오버랩)

### 검색 시스템
- **ChromaDB**: 벡터 데이터베이스 (코사인 유사도)

### 평가 지표

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

### 지표 해석 가이드
- **0.0 ~ 0.3**: 낮은 성능 (개선 필요)
- **0.3 ~ 0.6**: 보통 성능 (추가 최적화 권장)
- **0.6 ~ 0.8**: 좋은 성능 (실용적 수준)
- **0.8 ~ 1.0**: 매우 좋은 성능 (우수한 시스템)

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

### 베이스라인 실험 결과 (2025-09-23)

**실험 설정**: text-embedding-ada-002 + no_chunk + ChromaDB + GPT-4o-mini
**데이터**: 1,473개 문서, 575개 쿼리
**소요시간**: 813초 (약 13.5분)

```json
{
  "evaluation_results": [
    {"metric": "recall@1", "score": 0.0052},
    {"metric": "recall@3", "score": 0.0122},
    {"metric": "recall@5", "score": 0.0296},
    {"metric": "recall@10", "score": 0.0591},
    {"metric": "precision@1", "score": 0.0052},
    {"metric": "precision@3", "score": 0.0041},
    {"metric": "precision@5", "score": 0.0059},
    {"metric": "precision@10", "score": 0.0059},
    {"metric": "mrr", "score": 0.0161},
    {"metric": "map", "score": 0.0161},
    {"metric": "ndcg@1", "score": 0.0052},
    {"metric": "ndcg@3", "score": 0.0115},
    {"metric": "ndcg@5", "score": 0.0195},
    {"metric": "ndcg@10", "score": 0.0296}
  ]
}
```

**결과 해석**: 베이스라인 성능은 예상보다 낮게 나왔습니다. 이는 청킹 없이 전체 문서를 사용하여 노이즈가 많고, 현재 설정이 최적화되지 않았기 때문입니다. 향후 청킹 전략과 다른 임베딩 모델을 통한 성능 개선이 필요합니다.

## 🔍 토큰 제한 문제 해결

### 문제 상황
- 일부 테스트 쿼리가 OpenAI 토큰 제한(8,192) 초과
- 특히 수강 이력이 긴 쿼리들에서 발생

### 해결 방법
자동 수강 이력 트리밍 구현:
1. **토큰 수 체크**: tiktoken을 사용한 정확한 토큰 계산
2. **스마트 트리밍**: 기본 정보 유지, 수강 과목만 뒤에서부터 제거
3. **최소 보장**: 최소 5개 과목은 항상 유지
4. **로깅**: 트리밍 과정과 결과를 상세히 기록

### 트리밍 로그 예시
```
토큰 초과 감지 (12126), 수강 이력 트리밍 시도...
트리밍 성공: 12126 → 7438 토큰
```

## 📊 데이터 소스

### S3 데이터
- **버킷**: `career-hi`
- **PDF 경로**: `initial-dataset/pdf/` (1,473개 파일)
- **JSON 경로**: `initial-dataset/json/` (1,473개 파일)
- **페이지네이션**: 모든 S3 객체 자동 처리

### Ground Truth
- **테스트 쿼리**: 575개 (실제 사용자 프로필 기반)
- **Ground Truth**: 각 쿼리별 관련 채용공고 ID 매핑

## 🔄 Ground Truth 데이터 관리

### 새로운 GT CSV → JSONL 변환

새로운 Ground Truth CSV 파일을 받았을 때 실험에 사용할 JSONL 형태로 변환하는 유틸리티:

```bash
# 대화형 변환 (컬럼 매핑을 자동 감지하고 확인)
python utils/gt_converter.py new_ground_truth.csv data/test_queries_new.jsonl

# 컬럼 매핑 미리 지정
python utils/gt_converter.py input.csv output.jsonl --mapping "query:query_text,ground_truth_docs:relevant_doc_ids"

# 기존 JSONL 파일 검증
python utils/gt_converter.py --validate data/test_queries.jsonl
```

### 지원하는 CSV 형식

**필수 컬럼:**
- `query` 또는 `query_text`: 사용자 쿼리 텍스트
- `ground_truth_docs` 또는 `gt_docs`: 관련 문서 ID들 (쉼표 구분 또는 JSON 배열)

**선택적 컬럼:**
- `major`: 전공
- `interest_job`: 관심 직무 (쉼표 구분)
- `courses`: 수강 이력 (쉼표 구분)
- `certification`: 자격증 (쉼표 구분)
- `club_activities`: 동아리/대외활동 (쉼표 구분)
- `rec_idx`: 채용공고 인덱스
- `company_name`: 회사명
- `job_title`: 채용공고 제목

### 변환 결과 예시

**입력 CSV:**
```csv
query_text,gt_docs,major,interest_job
"생명공학 연구원 지원합니다","50436465,50123456",생명공학,"생명공학 연구원,제약회사 연구개발"
```

**출력 JSONL:**
```json
{"query": "생명공학 연구원 지원합니다", "ground_truth_docs": ["50436465", "50123456"], "user_profile": {"major": "생명공학", "interest_job": ["생명공학 연구원", "제약회사 연구개발"]}}
```

## 🐛 문제 해결

### 일반적인 문제들

**1. Docker 실행 실패**
```bash
# Docker Desktop 실행 확인
docker ps

# 권한 문제 시
sudo docker compose build
```

**2. API 키 오류**
```bash
# .env 파일 확인
cat .env | grep -E "(OPENAI|AWS)"
```

**3. 임베딩 캐시 문제**
```bash
# 캐시 삭제 후 재실행
rm -rf cache/embeddings/text_embedding_ada_002_no_chunk
./run_experiment.sh configs/baseline.yaml
```

**4. 메모리 부족**
```bash
# Docker 메모리 설정 확인 (최소 8GB 권장)
docker system info | grep Memory
```

### 실험 실패 시 체크리스트
- [ ] .env 파일에 올바른 API 키 설정
- [ ] Docker Desktop 실행 중
- [ ] AWS S3 접근 권한 확인
- [ ] 충분한 디스크 공간 (캐시용)
- [ ] 네트워크 연결 상태

## 🔮 향후 계획

### 추가 예정 기능
- [ ] **더 많은 임베딩 모델**: Sentence-BERT, Cohere 등
- [ ] **고급 청킹 전략**: Semantic chunking, Token-based chunking
- [ ] **다양한 검색 알고리즘**: FAISS, Elasticsearch
- [ ] **LLM 품질 평가**: 생성된 답변의 품질 측정
- [ ] **하이퍼파라미터 자동 튜닝**: Optuna 기반 최적화

### 실험 확장 방향
- [ ] **A/B 테스트**: 다양한 설정 조합의 성능 비교
- [ ] **사용자 피드백 통합**: 실제 사용자 만족도 측정
- [ ] **실시간 성능 모니터링**: 프로덕션 환경 성능 추적

## 📚 참고 자료

- [Docker 공식 문서](https://docs.docker.com/)
- [OpenAI Embeddings API](https://platform.openai.com/docs/guides/embeddings)
- [ChromaDB 문서](https://docs.trychroma.com/)
- [LangChain Text Splitters](https://python.langchain.com/docs/modules/data_connection/document_transformers/)
- [RAG 평가 지표 가이드](https://docs.ragas.io/en/stable/concepts/metrics/)

## 📝 라이센스

이 프로젝트는 Career-HY 팀의 내부 실험용으로 개발되었습니다.

---

**문의사항이나 개선 제안은 팀 슬랙 채널로 연락주세요!** 🚀