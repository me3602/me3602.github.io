---
layout: post
title: "GraphRAG 기술 리서치"
date: 2025-01-02
---

## 목차

1. [정의](#1-정의)
2. [사용 목적](#2-사용-목적)
3. [구성 요소](#3-구성-요소)
4. [기술적 장단점](#4-기술적-장단점)
5. [관련 라이브러리 및 오픈소스](#5-관련-라이브러리-및-오픈소스)
6. [활용 예시](#6-활용-예시)
7. [연관 기술](#7-연관-기술)
8. [결론](#8-결론)
9. [참고문헌](#9-참고문헌)

---

## 1. 정의

**GraphRAG(Graph Retrieval-Augmented Generation)**는 Microsoft Research에서 2024년에 발표한 차세대 RAG 기술로, 기존의 벡터 유사도 기반 RAG에 **지식 그래프(Knowledge Graph)**와 **커뮤니티 탐지(Community Detection)** 기법을 결합한 검색 증강 생성 방식이다.

기존 RAG(Retrieval-Augmented Generation)는 사용자의 질의와 유사한 텍스트 청크를 벡터 검색으로 찾아 LLM에 컨텍스트로 제공하는 방식이다. 이 방식은 특정 사실을 찾는 **지역적(Local) 질의**에는 효과적이지만, 문서 전체를 아우르는 **전역적(Global) 질의**—예를 들어 "이 데이터셋의 주요 주제는 무엇인가?"와 같은 요약형 질문—에는 한계가 있다.

GraphRAG는 이 문제를 해결하기 위해 LLM을 활용하여 원본 텍스트에서 엔티티(Entity)와 관계(Relationship)를 추출하고, 이를 지식 그래프로 구성한 뒤, 그래프 커뮤니티 탐지 알고리즘을 적용하여 **계층적 요약(Hierarchical Summarization)**을 생성한다. 이를 통해 지역적 질의와 전역적 질의 모두에 높은 품질의 응답을 제공한다.

---

## 2. 사용 목적

### 2.1 기존 RAG의 한계 극복

기존 RAG는 다음과 같은 상황에서 성능이 저하된다:

- **전역적 질의(Global Query)**: "이 문서 전체의 핵심 주제를 요약해줘"와 같이 데이터셋 전반에 걸친 이해가 필요한 질문
- **분산된 정보 연결**: 서로 다른 문서에 흩어져 있는 관련 정보를 종합하여 답변해야 하는 경우
- **다중 홉 추론(Multi-hop Reasoning)**: A→B→C와 같이 여러 단계의 추론이 필요한 질문

### 2.2 GraphRAG의 목적

GraphRAG는 다음을 목표로 설계되었다:

- **전역적 질의 지원**: 문서 전체를 아우르는 요약 및 주제 분석 질의에 대한 고품질 응답 생성
- **정보 연결성 강화**: 지식 그래프를 통해 엔티티 간의 관계를 명시적으로 모델링하여 분산된 정보를 효과적으로 연결
- **포괄성(Comprehensiveness) 향상**: 단순 유사도 검색 대비 더 포괄적이고 다양한 관점을 포함하는 응답 생성
- **계층적 이해**: 커뮤니티 구조를 통해 데이터를 다양한 추상화 수준에서 이해하고 탐색

---

## 3. 구성 요소

GraphRAG는 크게 **인덱싱 파이프라인(Indexing Pipeline)**과 **쿼리 엔진(Query Engine)**으로 구성된다.

### 3.1 인덱싱 파이프라인

인덱싱 파이프라인은 원본 문서를 지식 그래프와 커뮤니티 요약으로 변환하는 과정으로, 다음 6단계로 이루어진다:

#### Phase 1: 텍스트 유닛 분할 (Text Chunking)
원본 문서를 일정 크기의 텍스트 유닛(Text Unit)으로 분할한다. 각 텍스트 유닛은 이후 엔티티 추출의 기본 단위가 된다.

#### Phase 2: 그래프 추출 (Graph Extraction)
LLM을 사용하여 각 텍스트 유닛에서 **엔티티(Entity)**와 **관계(Relationship)**를 추출한다. 예를 들어, "삼성전자가 AI 반도체를 개발했다"라는 문장에서 `삼성전자`(조직), `AI 반도체`(기술)라는 엔티티와 `개발했다`라는 관계를 추출한다.

#### Phase 3: 그래프 보강 (Graph Augmentation)
추출된 그래프에 대해 **Leiden 알고리즘**을 적용하여 커뮤니티를 탐지한다. Leiden 알고리즘은 그래프에서 밀접하게 연결된 노드 그룹(커뮤니티)을 계층적으로 식별하는 알고리즘이다. 이를 통해 관련 엔티티들이 자연스럽게 주제별로 그룹화된다.

#### Phase 4: 커뮤니티 요약 (Community Summarization)
탐지된 각 커뮤니티에 대해 LLM을 사용하여 요약을 생성한다. 이 요약은 해당 커뮤니티에 속한 엔티티들과 그 관계를 종합적으로 설명하며, 전역적 질의에 대한 응답의 기반이 된다.

#### Phase 5: 임베딩 생성 (Embedding Generation)
텍스트 유닛, 엔티티, 관계, 커뮤니티 요약 등에 대한 벡터 임베딩을 생성하여 유사도 검색에 활용할 수 있도록 한다.

#### Phase 6: 문서-그래프 연결 (Document-Graph Linking)
원본 문서와 생성된 그래프 요소 간의 연결 관계를 저장하여, 응답 생성 시 출처 추적(Provenance)이 가능하도록 한다.

### 3.2 쿼리 엔진

GraphRAG는 질의 유형에 따라 다양한 검색 모드를 제공한다:

#### Global Search (전역 검색)
커뮤니티 요약을 활용하는 **Map-Reduce** 방식의 검색이다. 먼저 각 커뮤니티 요약에 대해 질의와의 관련성을 평가(Map)한 뒤, 관련 있는 요약들을 종합하여 최종 응답을 생성(Reduce)한다. "이 데이터셋의 주요 주제는?" 같은 전역적 질의에 적합하다.

#### Local Search (지역 검색)
질의와 관련된 엔티티를 먼저 식별한 뒤, 해당 엔티티의 이웃 노드, 관련 텍스트 유닛, 커뮤니티 보고서 등을 종합하여 컨텍스트를 구성한다. 특정 엔티티에 대한 상세 질의에 적합하다.

#### DRIFT Search (동적 추론 검색)
Local Search를 확장한 방식으로, 초기 검색 결과를 기반으로 후속 질의를 동적으로 생성하여 반복적으로 정보를 탐색한다. 커뮤니티 정보를 활용하여 검색 범위를 점진적으로 확장하며, 복잡한 다중 홉 질의에 효과적이다.

---

## 4. 기술적 장단점

### 4.1 장점

| 항목 | 설명 |
|------|------|
| **전역적 질의 지원** | 기존 RAG가 처리하지 못하는 요약형·분석형 질의에 대해 고품질 응답 생성 가능 |
| **포괄성(Comprehensiveness)** | 벤치마크에서 기존 RAG 대비 응답의 포괄성과 다양성이 유의미하게 향상됨 |
| **정보 연결성** | 지식 그래프를 통해 서로 다른 문서에 분산된 정보를 효과적으로 연결 |
| **계층적 이해** | 커뮤니티 구조를 통해 데이터를 다양한 추상화 수준에서 탐색 가능 |
| **출처 추적** | 그래프 구조를 통해 응답의 근거가 되는 원본 텍스트까지 추적 가능 |
| **다중 홉 추론** | 엔티티 간 관계를 명시적으로 모델링하여 복잡한 추론 질의 처리 가능 |

### 4.2 단점

| 항목 | 설명 |
|------|------|
| **높은 인덱싱 비용** | 엔티티 추출, 커뮤니티 요약 등에 다수의 LLM 호출이 필요하여 기존 RAG 대비 인덱싱 비용이 크게 증가 |
| **긴 인덱싱 시간** | 대규모 문서에 대한 그래프 구축에 상당한 시간이 소요됨 |
| **LLM 의존성** | 엔티티 추출 품질이 LLM 성능에 크게 의존하며, 추출 오류가 그래프 전체에 전파될 수 있음 |
| **복잡한 파이프라인** | 기존 RAG 대비 구성 요소가 많아 구축·운영·디버깅이 복잡함 |
| **단순 질의 시 과잉** | 단순 사실 확인 질의에는 기존 RAG로 충분하며, GraphRAG는 불필요한 오버헤드를 발생시킬 수 있음 |
| **그래프 업데이트 어려움** | 문서가 추가·변경될 때 지식 그래프를 증분 업데이트하는 것이 벡터 인덱스 업데이트보다 복잡함 |

---

## 5. 관련 라이브러리 및 오픈소스

### 5.1 Microsoft GraphRAG

- **GitHub**: [https://github.com/microsoft/graphrag](https://github.com/microsoft/graphrag)
- **Stars**: 31,900+
- **설명**: Microsoft Research에서 공식 발표한 GraphRAG 참조 구현체. Python 기반이며, 전체 인덱싱 파이프라인과 Global/Local/DRIFT 검색 모드를 지원한다. Azure OpenAI 및 OpenAI API와 통합되어 있다.
- **특징**: 가장 완성도 높은 구현체이나, 인덱싱 비용이 높은 편이다.

### 5.2 LightRAG

- **GitHub**: [https://github.com/HKUDS/LightRAG](https://github.com/HKUDS/LightRAG)
- **Stars**: 31,000+
- **설명**: 홍콩대학교에서 개발한 경량화된 GraphRAG 구현체. 그래프 기반 텍스트 인덱싱과 듀얼 레벨 검색(Low-level, High-level)을 지원하며, Microsoft GraphRAG 대비 더 낮은 비용으로 유사한 성능을 달성한다.
- **특징**: 빠른 인덱싱, 증분 업데이트 지원, 다양한 LLM 백엔드 호환.

### 5.3 nano-graphrag

- **GitHub**: [https://github.com/gusye1234/nano-graphrag](https://github.com/gusye1234/nano-graphrag)
- **Stars**: 3,800+
- **설명**: Microsoft GraphRAG의 핵심 기능을 약 800줄의 코드로 구현한 경량 버전. 학습 및 프로토타이핑 목적에 적합하다.
- **특징**: 간결한 코드베이스, 커스터마이징 용이, 빠른 실험 가능.

### 5.4 fast-graphrag

- **GitHub**: [https://github.com/circlemind-ai/fast-graphrag](https://github.com/circlemind-ai/fast-graphrag)
- **Stars**: 3,800+
- **설명**: Circlemind AI에서 개발한 고성능 GraphRAG 구현체. Microsoft GraphRAG 대비 약 6배 낮은 인덱싱 비용을 주장하며, PageRank 기반의 그래프 탐색을 활용한다.
- **특징**: 비용 효율성, 증분 업데이트 지원, 빠른 인덱싱.

### 5.5 LlamaIndex GraphRAG

- **GitHub**: [https://github.com/run-llama/llama_index](https://github.com/run-llama/llama_index)
- **설명**: LlamaIndex 프레임워크 내에서 GraphRAG 패턴을 구현할 수 있는 모듈을 제공한다. `PropertyGraphIndex`와 `KnowledgeGraphIndex`를 통해 그래프 기반 RAG를 구축할 수 있다.
- **특징**: 기존 LlamaIndex 생태계와의 통합, 다양한 그래프 스토어 백엔드 지원.

---

## 6. 활용 예시

### 6.1 대규모 문서 분석

법률 문서, 특허 문서, 학술 논문 등 대규모 텍스트 코퍼스에서 핵심 주제를 추출하고, 문서 간 관계를 파악하는 데 활용된다. 예를 들어, 수천 건의 특허 문서에서 기술 트렌드를 분석하거나, 법률 판례 간의 인용 관계를 탐색할 수 있다.

### 6.2 기업 내부 지식 관리

기업의 내부 문서(위키, 보고서, 이메일 등)를 지식 그래프로 구조화하여, "우리 회사에서 AI 관련 프로젝트를 진행한 팀은?" 같은 전역적 질의에 대한 포괄적인 답변을 제공할 수 있다.

### 6.3 뉴스 및 소셜 미디어 분석

뉴스 기사나 소셜 미디어 데이터에서 인물, 조직, 이벤트 간의 관계를 추출하고, 특정 이슈에 대한 전체적인 맥락을 파악하는 데 활용된다.

### 6.4 과학 연구 지원

연구 논문에서 연구자, 기관, 연구 주제 간의 관계를 그래프로 구성하여, 특정 분야의 연구 동향을 파악하거나 잠재적 협력 기회를 발견하는 데 활용할 수 있다.

### 6.5 고객 지원 시스템

제품 매뉴얼, FAQ, 고객 문의 이력 등을 지식 그래프로 구성하여, 복잡한 기술 지원 질의에 대해 관련 정보를 종합적으로 제공하는 챗봇 시스템에 활용된다.

---

## 7. 연관 기술

### 7.1 지식 그래프 (Knowledge Graph)

GraphRAG의 핵심 데이터 구조로, 엔티티를 노드로, 관계를 엣지로 표현하는 그래프 구조이다. Neo4j, Amazon Neptune 등의 그래프 데이터베이스와 함께 사용되며, RDF, Property Graph 등의 모델이 활용된다.

### 7.2 벡터 데이터베이스 (Vector Database)

임베딩 벡터를 저장하고 유사도 검색을 수행하는 데이터베이스로, GraphRAG에서도 엔티티 및 텍스트 유닛의 임베딩을 저장하는 데 활용된다. 대표적으로 Pinecone, Weaviate, Milvus, Chroma, Qdrant 등이 있다.

### 7.3 대규모 언어 모델 (LLM)

GraphRAG의 모든 단계에서 핵심적으로 사용된다. 엔티티 추출, 커뮤니티 요약 생성, 최종 응답 생성 등에 GPT-4, Claude, Llama 등의 LLM이 활용된다.

### 7.4 커뮤니티 탐지 알고리즘

GraphRAG에서는 **Leiden 알고리즘**이 핵심적으로 사용된다. Leiden 알고리즘은 Louvain 알고리즘의 개선 버전으로, 그래프에서 모듈성(Modularity)을 최적화하여 밀접하게 연결된 노드 그룹을 계층적으로 탐지한다. 이를 통해 지식 그래프의 엔티티들이 주제별로 자연스럽게 클러스터링된다.

### 7.5 기존 RAG (Retrieval-Augmented Generation)

GraphRAG의 기반이 되는 기술로, 벡터 유사도 검색을 통해 관련 문서 청크를 찾아 LLM의 컨텍스트로 제공하는 방식이다. LangChain, LlamaIndex 등의 프레임워크에서 널리 구현되어 있다.

### 7.6 NER (Named Entity Recognition)

텍스트에서 엔티티를 식별하는 자연어 처리 기술로, GraphRAG의 엔티티 추출 단계와 밀접하게 관련된다. GraphRAG에서는 전통적인 NER 모델 대신 LLM을 활용하여 더 유연한 엔티티 추출을 수행한다.

### 7.7 Graph Neural Network (GNN)

그래프 구조 데이터에 대한 딥러닝 기법으로, 향후 GraphRAG의 그래프 임베딩이나 커뮤니티 탐지 단계에서 활용될 가능성이 있다.

---

## 8. 결론

GraphRAG는 기존 RAG의 근본적인 한계—전역적 질의에 대한 취약성과 분산된 정보의 연결 부재—를 지식 그래프와 커뮤니티 탐지를 통해 해결한 혁신적인 기술이다.

Microsoft Research의 벤치마크에 따르면, GraphRAG는 기존 RAG 대비 응답의 **포괄성(Comprehensiveness)**과 **다양성(Diversity)**에서 유의미한 향상을 보여주었다. 특히 "데이터셋의 주요 주제는 무엇인가?"와 같은 전역적 질의에서 기존 RAG가 의미 있는 응답을 생성하지 못하는 반면, GraphRAG는 커뮤니티 요약을 활용하여 포괄적인 답변을 제공한다.

다만, 높은 인덱싱 비용과 복잡한 파이프라인은 실제 프로덕션 환경에서의 도입 장벽이 될 수 있다. 이를 해결하기 위해 LightRAG, fast-graphrag 등 비용 효율적인 대안 구현체들이 등장하고 있으며, 오픈소스 생태계가 빠르게 성장하고 있다.

GraphRAG는 단순한 질의응답을 넘어, 대규모 텍스트 데이터에 대한 **구조적 이해**와 **전역적 분석**이 필요한 모든 영역에서 강력한 도구가 될 것이다. 기존 RAG와 GraphRAG를 질의 유형에 따라 적절히 조합하는 **하이브리드 접근법**이 현실적인 최적 전략이 될 것으로 전망된다.

---

## 9. 참고문헌

1. Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, Jonathan Larson. "From Local to Global: A Graph RAG Approach to Query-Focused Summarization." *arXiv preprint arXiv:2404.16130*, 2024. [https://arxiv.org/abs/2404.16130](https://arxiv.org/abs/2404.16130)

2. Microsoft Research. "GraphRAG: Unlocking LLM discovery on narrative private data." Microsoft Research Blog, 2024. [https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/](https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/)

3. Microsoft GraphRAG GitHub Repository. [https://github.com/microsoft/graphrag](https://github.com/microsoft/graphrag)

4. Microsoft GraphRAG Documentation. [https://microsoft.github.io/graphrag/](https://microsoft.github.io/graphrag/)

5. Zirui Guo, Lianghao Xia, Yanhua Yu, Tu Ao, Chao Huang. "LightRAG: Simple and Fast Retrieval-Augmented Generation." *arXiv preprint arXiv:2410.05779*, 2024. [https://arxiv.org/abs/2410.05779](https://arxiv.org/abs/2410.05779)

6. V.A. Traag, L. Waltman, N.J. van Eck. "From Louvain to Leiden: guaranteeing well-connected communities." *Scientific Reports*, 9(1), 5233, 2019. [https://arxiv.org/abs/1810.08473](https://arxiv.org/abs/1810.08473)

7. Patrick Lewis, Ethan Perez, Aleksandra Piktus, et al. "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks." *NeurIPS*, 2020. [https://arxiv.org/abs/2005.11401](https://arxiv.org/abs/2005.11401)
