# LangGraph-Graph RAG Backend Overview

LangGraph-Graph RAG(Backend)는 **LangGraph + SGLang** 조합으로 동작하는 Vector-Graph Hybrid RAG 플랫폼입니다. 입력 문서를 LangGraph Upload 파이프라인으로 구조화하고, 질의 시 Hop 분류 결과에 따라 Vector Retrieval / Weaviate GraphRAG / Legacy GraphDB(Neo4j) 경로를 자동 선택해 답변을 생성합니다.

---

## 핵심 기능

- **LangGraph 기반 파이프라인**: 업로드/인덱싱과 질의/추론 모두 LangGraph 상태 머신으로 구성.
- **3-Way Retrieval**: Hop ≤2 → Vector(Weaviate), 3~5 → Weaviate GraphRAG, ≥6 → Neo4j GraphDB.
- **LLM 라우터**: Small LLM Hop Classifier + 휴리스틱으로 Reasoning Depth 추정.
- **SGLang 모델 서버**: Generator/Embedding/Reranker/Summarizer를 GPU 별로 상시 제공, LazyModelManager로 자동 관리.
- **자동 그래프 업서트**: OCR→LLM 추출 결과를 Weaviate와 Neo4j에 동시에 업서트.
- **비동기 작업·모니터링**: 업로드/OCR/임베딩 진행 상황을 task API로 추적

---

## 백엔드 아키텍처

```
┌──────────────┐    ┌────────────────────────┐
│ Input Layer  │ → │ LangGraph Upload Flow  │ → Markdown + graph JSON
└──────────────┘    └────────────────────────┘
                              │
┌──────────────┐    ┌────────────────────────┐    ┌───────────────────┐
│ User Query   │ → │ LangGraph RAG Workflow │ → │ SGLang Generation │
└──────────────┘    └────────────────────────┘    └───────────────────┘
                              │
                 ┌────────────┴────────────┐
                 │ 3-Way Retrieval Router │
                 └────────────┬────────────┘
                              │
      ┌───────────────────────┼────────────────────────┐
      │                       │                        │
  Path 1                 Path 2                   Path 3
  Vector ≤2 hop          Weaviate GraphRAG        Neo4j GraphDB ≥6 hop
```

### 입력/전처리 레이어 (LangGraphUploadPipeline)
1. **Conversion & Layout**: `run_file_processor.py`가 PDF/Office/이미지/음성을 처리해 `Results/1.Converted_images` 생성.
2. **OCR & Markdown**: `run_ocr_processing()` + vLLM 기반 OCR Flux로 페이지 단위 Markdown 산출.
3. **LLM 메타데이터 추출**: `LLMMetadataExtractor`가 Markdown을 청킹해 Embedding 모델로 엔티티/이벤트/관계 JSON(`*.graph.json`) 저장.
4. **업서트**:
   - `GraphSchemaManager` → Weaviate(Entity/Event/Relation 클래스를 자동 보장 후 업서트)
   - `LegacyGraphIngestor` → Neo4j에 노드/관계 MERGE 및 constraint 보장
5. **Late Chunking & Embedding**: `embedding_text.py`가 Markdown을 Weaviate TextDocument 컬렉션에 업서트.

### 질의/추론 레이어 (RAGPipeline)
1. **Hop Classifier** (`hop_classifier.py`): Small LLM이 질의 복잡도를 추정.
2. **LangGraph Workflow** (`rag_pipeline.py`): `planner → router → retrieval node → generation → finalizer` 순으로 상태 관리.
3. **Retrieval Paths**:
   - **Path 1: Vector RAG** – `embedding_text.text_search` + `text_reranker`.
   - **Path 2: Weaviate GraphReasoner** – `graph_reasoner.py`가 LangGraph 서브그래프로 다중 hop 확장 및 context snippet 생성.
   - **Path 3: Legacy GraphDB** – `legacy_graph_client.py`가 Cypher 템플릿으로 Neo4j 경로를 수집.
4. **Generation** (`generator.py`): SGLang LLM이 검색 컨텍스트 + 오리지널 질의를 기반으로 답변 생성, 필요 시 `refiner.py`/`evaluator.py`로 후처리

### 주요 모듈 맵

```
backend/
├── api/               # FastAPI 라우터 및 백그라운드 작업
├── notebooklm/
│   ├── rag_pipeline.py        # LangGraph RAG 워크플로우
│   ├── graph_reasoner.py      # Weaviate 그래프 확장
│   ├── legacy_graph_client.py # Neo4j 경로 조회
│   ├── legacy_graph_ingestor.py # Neo4j 업서트
│   ├── embedding_text.py / embedding_image.py
│   ├── shared_embedding.py    # SGLang 임베딩/리랭커 클라이언트
│   ├── generator.py / refiner.py / evaluator.py
│   └── config.py              # 모델·경로·Graph 설정
└── nextits_data/
    └── pipe/
        ├── langgraph_upload_pipeline.py
        ├── run_file_processor.py (convert/layout/OCR)
        └── ocr_pipe/ ···
```

---

## 데이터 파이프라인 요약

1. **파일 업로드 (`POST /upload/files`)**
   - `api/routes.py`가 파일을 세션 디렉터리에 저장 후 `run_processing_pipeline`을 큐에 등록.
2. **GPU Task Queue**
   - `run_file_processor.py --stage convert/layout/ocr` 순으로 실행.
3. **LangGraph 업로드 워크플로우**
   - OCR 결과 수집 → `LLMMetadataExtractor` 실행 → `GraphSchemaManager` + `LegacyGraphIngestor` 업서트 → `embedding_text` Late Chunking.
4. **Weaviate & Neo4j 상태**
   - Weaviate: TextDocument + GraphEntity/Event/Relation 컬렉션 유지.
   - Neo4j: Entity/Event 노드 및 relation edge를 uuid 기반으로 MERGE.

---

## 질의 처리 플로우

1. `POST /v1/chat` → `rag_pipeline.retrieve()` 호출.
2. Hop 분류 결과에 따라 Router가 Path 1/2/3 중 선택.
3. 각 Path는 컨텍스트 스니펫을 `state.context_snippets`에 push.
4. `generator.py`가 오리지널 질의 + 컨텍스트로 답변 생성.
5. (옵션) `refiner.py`가 답변을 정제, `evaluator.py`가 품질 노트를 남김.
6. 응답에는 plan/hops/notes/context_snippets가 포함돼 디버깅에 활용.

---

## 설치 및 실행 (Backend)

```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 환경 변수 (.env 예시)
export GRAPH_RAG_ENABLED=true
export LEGACY_GRAPH_ENABLED=true
export LEGACY_GRAPH_URI=bolt://localhost:7687
export LEGACY_GRAPH_USER=neo4j
export LEGACY_GRAPH_PASSWORD=your_password

python main.py
# SGLang 임베딩/리랭커 서버는 최초 요청 시 자동 기동됩니다.
```

Neo4j를 로컬 테스트로 띄우려면:

```bash
docker run -d --name neo4j-dev \
  -p7474:7474 -p7687:7687 \
  -e NEO4J_AUTH=neo4j/your_password \
  -v /home/nextits/neo4j-data:/data \
  neo4j:5
```

---

## 주요 설정 (`config.py`)

- **모델**: `EMBEDDING_MODEL=Qwen/Qwen3-Embedding-4B`, `LLM_MODEL=Qwen/Qwen3-0.6B`, `HOP_CLASSIFIER_MODEL=Qwen/Qwen3-0.6B` 등.
- **Graph RAG 토글**: `GRAPH_RAG_ENABLED`, `LANGGRAPH_ENABLED`, `GOT_MODE_ENABLED`, `GRAPH_MAX_HOPS`.
- **Legacy Graph**: `LEGACY_GRAPH_ENABLED`, `LEGACY_GRAPH_URI`, `LEGACY_GRAPH_LABELS`, `LEGACY_GRAPH_MAX_PATHS`.
- **Weaviate**: `WEAVIATE_HOST/PORT`, `WEAVIATE_TEXT_CLASS`, `WEAVIATE_VECTORIZER`.
- **세션 디렉터리**: `DATA_ROOT/Results`, `sessions/<id>` 구조.

---

## API 하이라이트

- `POST /upload/files` – 파일 업로드 및 LangGraph 업로드 파이프라인 실행.
- `POST /v1/chat` – 3-Way RAG 질의, plan/notes/context를 함께 반환.
- `GET /task_status/{task_id}` – 업로드 작업 진행 상황.
- `GET /files`, `GET /files/{filename}` – 세션 단위 문서/결과 확인.

---

## 로깅 & 운영

- `sglang_embedding_server.log`, `sglang_reranker_server.log` – 모델 서버 상태.
- `Results/graph_metadata/*.graph.json` – LLM 추출 결과 아카이브.
- `LegacyGraphIngestor`는 최초 호출 시 constraint를 자동 생성하므로 별도 수동 작업이 필요 없습니다.
- GPU 자원은 LazyModelManager가 idle 20초 후 자동 해제합니다.

---

## 향후 작업 로드맵

1. **GoT(Graph of Thought) 모드 실험**
   - `GOT_MODE_ENABLED` 토글을 활성화하고 LangGraph 플로우에 GoT 브랜치를 통합해 단계별 reasoning state를 축적.
2. **Hop 분류기 고도화**
   - Small LLM 기반 분류기의 threshold/feature를 튜닝하고, 질의 메타데이터(토큰 길이, 개체 수 추정)를 결합한 Hybrid Router 연구.
3. **다중 그래프 검색 최적화**
   - Weaviate 3~5-hop 확장 시 컨텍스트 필터링/중복 제거 로직 개선 및 Neo4j 6-hop 이상 경로 탐색 시 Cypher 템플릿을 추가.
4. **LangGraph 워크플로우 모니터링**
   - 노드별 latency/에러율을 Prometheus 지표로 수집하고, 실패 재시도 정책을 GraphReasoner/LegacyGraphClient에 통합.

---

## 기여 & 문의

- jeongnext@hnextits.com


이슈/PR을 환영하며, 백엔드 모듈 중심의 개선 사항을 자유롭게 제안해 주세요. 

