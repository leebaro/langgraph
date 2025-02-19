---
title: 튜토리얼 (Tutorials)
---

# 튜토리얼 (Tutorials)

LangGraph 또는 LLM 앱 개발이 처음이신가요? 이 자료를 읽고 첫 번째 애플리케이션을 구축하여 실행하십시오.

## 시작하기 🚀 {#quick-start}

- [LangGraph 퀵스타트 (Quickstart)](introduction.ipynb): 도구를 사용하고 대화 기록을 추적할 수 있는 챗봇을 구축합니다. Human-in-the-loop (사람이 참여하는) 기능을 추가하고 타임 트래블 (time-travel)이 어떻게 작동하는지 살펴봅니다.
- [일반적인 워크플로우 (Common Workflows)](workflows/index.md): LangGraph로 구현된 LLM을 사용하는 가장 일반적인 워크플로우에 대한 개요입니다.
- [LangGraph 서버 퀵스타트 (Server Quickstart)](langgraph-platform/local-server.md): LangGraph 서버를 로컬에서 실행하고 REST API 및 LangGraph Studio Web UI를 사용하여 상호 작용합니다.
- [LangGraph 템플릿 퀵스타트 (Template Quickstart)](../concepts/template_applications.md): 템플릿 애플리케이션을 사용하여 LangGraph 플랫폼으로 빌드를 시작합니다.
- [LangGraph 클라우드 배포 퀵스타트 (Cloud Quickstart)](../cloud/quick_start.md): LangGraph 클라우드를 사용하여 LangGraph 앱을 배포합니다.

## 사용 사례 🛠️ {#use-cases}

특정 시나리오에 맞게 조정된 실제 구현을 살펴봅니다.

### 챗봇 (Chatbots)

- [고객 지원 (Customer Support)](customer-support/customer-support.ipynb): 항공편, 호텔 및 렌터카를 위한 다기능 지원 봇을 구축합니다.
- [사용자 요구 사항으로부터 프롬프트 생성 (Prompt Generation from User Requirements)](chatbots/information-gather-prompting.ipynb): 정보 수집 챗봇을 구축합니다.
- [코드 어시스턴트 (Code Assistant)](code_assistant/langgraph_code_assistant.ipynb): 코드 분석 및 생성 어시스턴트를 구축합니다.

### RAG (Retrieval Augmented Generation)

- [Agentic RAG](rag/langgraph_agentic_rag.ipynb): 에이전트를 사용하여 사용자 질문에 답변하기 전에 가장 관련성 높은 정보를 검색하는 방법을 알아냅니다.
- [Adaptive RAG](rag/langgraph_adaptive_rag.ipynb): Adaptive RAG는 (1) 쿼리 분석과 (2) 능동적/자체 수정 RAG를 결합하는 RAG 전략입니다. 구현: https://arxiv.org/abs/2403.14403
    - 로컬 LLM을 사용하는 버전: [Adaptive RAG using local LLMs](rag/langgraph_adaptive_rag_local.ipynb)
- [Corrective RAG](rag/langgraph_crag.ipynb): LLM을 사용하여 주어진 소스에서 검색된 정보의 품질을 평가하고 품질이 낮으면 다른 소스에서 정보를 검색하려고 시도합니다. 구현: https://arxiv.org/pdf/2401.15884.pdf
    - 로컬 LLM을 사용하는 버전: [Corrective RAG using local LLMs](rag/langgraph_crag_local.ipynb)
- [Self-RAG](rag/langgraph_self_rag.ipynb): Self-RAG는 검색된 문서 및 생성에 대한 자체 평가/자체 등급을 통합하는 RAG 전략입니다. 구현: https://arxiv.org/abs/2310.11511.
    - 로컬 LLM을 사용하는 버전: [Self-RAG using local LLMs](rag/langgraph_self_rag_local.ipynb)
- [SQL 에이전트 (SQL Agent)](sql-agent.ipynb): SQL 데이터베이스에 대한 질문에 답변할 수 있는 SQL 에이전트를 구축합니다.

### 에이전트 아키텍처 (Agent Architectures)

#### 멀티 에이전트 시스템 (Multi-Agent Systems)

- [네트워크 (Network)](multi_agent/multi-agent-collaboration.ipynb): 둘 이상의 에이전트가 작업에서 협업할 수 있도록 합니다.
- [감독자 (Supervisor)](multi_agent/agent_supervisor.ipynb): LLM을 사용하여 개별 에이전트를 오케스트레이션하고 위임합니다.
- [계층적 팀 (Hierarchical Teams)](multi_agent/hierarchical_agent_teams.ipynb): 에이전트의 중첩된 팀을 오케스트레이션하여 문제를 해결합니다.

#### 계획 에이전트 (Planning Agents)

- [계획 및 실행 (Plan-and-Execute)](plan-and-execute/plan-and-execute.ipynb): 기본 계획 및 실행 에이전트를 구현합니다.
- [관찰 없는 추론 (Reasoning without Observation)](rewoo/rewoo.ipynb): 관찰 내용을 변수로 저장하여 재계획을 줄입니다.
- [LLMCompiler](llm-compiler/LLMCompiler.ipynb): 플래너에서 작업의 DAG를 스트리밍하고 즉시 실행합니다.

#### 성찰 및 비평 (Reflection & Critique)

- [기본 성찰 (Basic Reflection)](reflection/reflection.ipynb): 에이전트에게 출력을 성찰하고 수정하도록 프롬프트합니다.
- [Reflexion](reflexion/reflexion.ipynb): 다음 단계를 안내하기 위해 누락되거나 불필요한 세부 정보를 비판합니다.
- [Tree of Thoughts](tot/tot.ipynb): 점수가 매겨진 트리를 사용하여 문제에 대한 후보 솔루션을 검색합니다.
- [Language Agent Tree Search](lats/lats.ipynb): 성찰과 보상을 사용하여 에이전트에 대한 몬테카를로 트리 검색을 유도합니다.
- [Self-Discover Agent](self-discover/self-discover.ipynb): 자체 기능에 대해 학습하는 에이전트를 분석합니다.

### 평가 (Evaluation)

- [에이전트 기반 (Agent-based)](chatbot-simulation-evaluation/agent-simulation-evaluation.ipynb): 시뮬레이션된 사용자 상호 작용을 통해 챗봇을 평가합니다.
- [LangSmith에서 (In LangSmith)](chatbot-simulation-evaluation/langsmith-agent-simulation-evaluation.ipynb): 대화 데이터 세트를 통해 LangSmith에서 챗봇을 평가합니다.

### 실험적 (Experimental)

- [웹 연구 (STORM)](storm/storm.ipynb): 연구 및 다각적 QA를 통해 위키피디아와 유사한 기사를 생성합니다.
- [TNT-LLM](tnt-llm/tnt-llm.ipynb): Microsoft에서 Bing Copilot 애플리케이션을 위해 개발한 분류 시스템을 사용하여 사용자 의도의 풍부하고 해석 가능한 분류 체계를 구축합니다.
- [웹 탐색 (Web Navigation)](web-navigation/web_voyager.ipynb): 웹 사이트를 탐색하고 상호 작용할 수 있는 에이전트를 구축합니다.
- [경쟁적 프로그래밍 (Competitive Programming)](usaco/usaco.ipynb): USA Computing Olympiad의 문제를 해결하기 위해 few-shot "episodic memory" 및 human-in-the-loop 협업을 사용하는 에이전트를 구축합니다. Shi, Tang, Narasimhan 및 Yao의 ["언어 모델이 올림피아드 프로그래밍을 해결할 수 있습니까? (Can Language Models Solve Olympiad Programming?)"](https://arxiv.org/abs/2404.10952v1) 논문에서 각색했습니다.
- [복잡한 데이터 추출 (Complex data extraction)](extraction/retries.ipynb): 함수 호출을 사용하여 복잡한 추출 작업을 수행할 수 있는 에이전트를 구축합니다.

## LangGraph 플랫폼 🧱 {#platform}

### 인증 및 액세스 제어 (Authentication & Access Control)

다음 3부 가이드에서 기존 LangGraph 플랫폼 배포에 사용자 지정 인증 및 권한 부여를 추가합니다.

1. [사용자 지정 인증 설정 (Setting Up Custom Authentication)](auth/getting_started.md): OAuth2 인증을 구현하여 배포에서 사용자를 인증합니다.
2. [리소스 권한 부여 (Resource Authorization)](auth/resource_auth.md): 사용자가 비공개 대화를 할 수 있도록 합니다.
3. [인증 공급자 연결 (Connecting an Authentication Provider)](auth/add_auth_server.md): 실제 사용자 계정을 추가하고 OAuth2를 사용하여 유효성을 검사합니다.