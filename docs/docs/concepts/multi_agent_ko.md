# 멀티 에이전트 시스템 (Multi-agent Systems)

<mark>[에이전트(agent)](./agentic_concepts.md#agent-architectures)는 _LLM을 사용하여 애플리케이션의 제어 흐름을 결정하는 시스템_</mark> 입니다. 이러한 시스템을 개발할 때 시간이 지남에 따라 복잡성이 증가하여 관리하고 확장하기가 더 어려워질 수 있습니다. 예를 들어 다음과 같은 문제에 직면할 수 있습니다.

- 에이전트가 너무 많은 도구를 마음대로 사용할 수 있어서 다음에 어떤 도구를 호출할지에 대한 결정을 제대로 내리지 못함
- 컨텍스트가 너무 복잡해져서 단일 에이전트가 추적하기 어려움
- 시스템에 여러 전문 분야가 필요함 (예: 플래너, 연구원, 수학 전문가 등)

이러한 문제를 해결하기 위해 <mark>애플리케이션을 여러 개의 더 작고 독립적인 에이전트로 분리하고 이를 결합하여 **멀티 에이전트 시스템(multi-agent system)** 을 구성하는 것을 고려할 수 있습니다.</mark> 이러한 독립적인 에이전트는 프롬프트와 LLM 호출처럼 간단할 수도 있고, [ReAct](./agentic_concepts.md#react-implementation) 에이전트처럼 복잡할 수도 있습니다 (그리고 훨씬 더 복잡할 수도 있습니다!).

멀티 에이전트 시스템을 사용하는 주요 이점은 다음과 같습니다.

- **모듈성(Modularity)**: 개별 에이전트를 통해 에이전트 시스템을 더 쉽게 개발, 테스트 및 유지 관리할 수 있습니다.
- **특수성(Specialization)**: 특정 도메인에 집중된 전문 에이전트를 생성하여 전체 시스템 성능을 향상시킬 수 있습니다.
- **제어(Control)**: 함수 호출에 의존하는 대신 에이전트가 통신하는 방식을 명시적으로 제어할 수 있습니다.

## 멀티 에이전트 아키텍처 (Multi-agent architectures)

![](./img/multi_agent/architectures.png)

멀티 에이전트 시스템에서 에이전트를 연결하는 방법에는 여러 가지가 있습니다.

- **네트워크(Network)**: 각 에이전트는 [다른 모든 에이전트](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/)와 통신할 수 있습니다. 모든 에이전트는 다음에 어떤 에이전트를 호출할지 결정할 수 있습니다.
- **감독자(Supervisor)**: 각 에이전트는 단일 [감독자(supervisor)](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/) 에이전트와 통신합니다. 감독자 에이전트는 다음에 어떤 에이전트를 호출해야 하는지 결정합니다.
- **감독자 (도구 호출)(Supervisor (tool-calling))**: 이는 감독자 아키텍처의 특수한 경우입니다. 개별 에이전트는 도구로 표현될 수 있습니다. 이 경우 감독자 에이전트는 도구 호출 LLM을 사용하여 어떤 에이전트 도구를 호출할지, 그리고 해당 에이전트에 전달할 인수를 결정합니다.
- **계층적(Hierarchical)**: [감독자의 감독자](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/)를 사용하여 멀티 에이전트 시스템을 정의할 수 있습니다. 이는 감독자 아키텍처의 일반화이며, 더 복잡한 제어 흐름을 허용합니다.
- **사용자 정의 멀티 에이전트 워크플로(Custom multi-agent workflow)**: 각 에이전트는 에이전트의 하위 집합하고만 통신합니다. 흐름의 일부는 결정적이며, 일부 에이전트만 다음에 어떤 에이전트를 호출할지 결정할 수 있습니다.

### 핸드오프 (Handoffs)

멀티 에이전트 아키텍처에서 에이전트는 그래프 노드(graph nodes)로 표현될 수 있습니다. 각 에이전트 노드는 자신의 단계를 실행하고 실행을 완료할지 또는 다른 에이전트로 라우팅할지를 결정합니다. 여기에는 자기 자신으로의 라우팅도 포함됩니다(예: 루프로 실행). 멀티 에이전트 상호작용에서 일반적인 패턴은 핸드오프(handoffs)입니다. 이는 한 에이전트가 다른 에이전트에게 제어권을 넘기는 것을 말합니다. 핸드오프를 통해 다음 사항을 지정할 수 있습니다:

- __목적지(destination)__: 이동할 대상 에이전트 (예: 이동할 노드의 이름)
[해당 에이전트에게 전달할 정보](#에이전트-간-통신-communication-between-agents)

LangGraph에서 핸드오프를 구현하기 위해, 에이전트 노드는 제어 흐름과 상태 업데이트를 모두 결합할 수 있는 [`Command`](./low_level.md#command) 객체를 반환할 수 있습니다:

```python
def agent(state) -> Command[Literal["agent", "another_agent"]]:
    # 라우팅/중단 조건은 LLM 도구 호출(LLM tool call), 구조화된 출력(structured output) 등 무엇이든 될 수 있습니다.
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    return Command(
        # Specify which agent to call next
        goto=goto,
        # Update the graph state
        update={"my_state_key": "my_state_value"}
    )
```

각 에이전트 노드 자체가 그래프(즉, [서브 그래프(subgraph)](./low_level.md#subgraphs))인 더 복잡한 시나리오에서, 에이전트 서브 그래프의 노드 중 하나가 다른 에이전트로 이동하려고 할 수 있습니다. 예를 들어 `alice`와 `bob`이라는 두 에이전트(상위 그래프의 서브 그래프 노드)가 있고 `alice`가 `bob`으로 이동해야 하는 경우, `Command` 객체에서 `graph=Command.PARENT`를 설정할 수 있습니다.

```python
def some_node_inside_alice(state)
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        # specify which graph to navigate to (defaults to the current graph)
        graph=Command.PARENT,
    )
```

!!! note "참고"
    `Command(graph=Command.PARENT)`를 사용하여 통신하는 서브 그래프(subgraphs)에 대한 시각화(visualization)를 지원해야 하는 경우, 다음과 같이 노드 함수(node function)로 래핑(wrap)하고 `Command` 어노테이션(annotation)을 추가해야 합니다.

    ```python
    builder.add_node(alice)
    ```

    다음과 같이 해야 합니다.

    ```python
    def call_alice(state) -> Command[Literal["bob"]]:
        return alice.invoke(state)

    builder.add_node("alice", call_alice)
    ```

#### 도구로서의 핸드오프 (Handoffs as tools)

가장 일반적인 에이전트 유형 중 하나는 ReAct 스타일의 도구 호출 에이전트입니다. 이러한 유형의 에이전트의 경우, 일반적인 패턴은 핸드오프를 도구 호출로 래핑하는 것입니다 (예):

```python
def transfer_to_bob(state):
    """Transfer to bob."""
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        graph=Command.PARENT,
    )
```

이는 상태 업데이트 외에도 제어 흐름이 포함된 도구에서 그래프 상태를 업데이트하는 특별한 경우입니다.

!!! important

    `Command`를 반환하는 도구를 사용하려면, 미리 빌드된 [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] / [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode] 컴포넌트를 사용하거나, 도구에서 반환된 `Command` 객체를 수집하고 해당 목록을 반환하는 자체 도구 실행 노드를 구현할 수 있습니다 (예:):
    
    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```

이제 다양한 멀티 에이전트 아키텍처를 자세히 살펴보겠습니다.

### 네트워크 (Network)

이 아키텍처에서 에이전트는 그래프 노드(graph nodes)로 정의됩니다. 각 에이전트는 다른 모든 에이전트(다대다 연결)와 통신할 수 있으며, 다음에 호출할 에이전트를 결정할 수 있습니다. <mark>이 아키텍처는 에이전트의 명확한 계층 구조나 에이전트를 호출해야 하는 특정 순서가 없는 문제에 적합합니다.</mark>

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def agent_1(state: MessagesState) -> Command[Literal["agent_2", "agent_3", END]]:
    # LLM에 상태의 관련 부분(예: state["messages"])을 전달하여 다음에 호출할 에이전트를 결정할 수 있습니다.
    # 일반적인 패턴은 모델을 구조화된 출력(structured output)으로 호출하는 것입니다 (예: "next_agent" 필드가 있는 출력을 반환하도록 강제).
    response = model.invoke(...)
    # LLM의 결정에 따라 에이전트 중 하나로 라우팅하거나 종료합니다.
    # LLM이 "__end__"를 반환하면 그래프 실행이 종료됩니다.
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_2(state: MessagesState) -> Command[Literal["agent_1", "agent_3", END]]:
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_3(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    ...
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
builder.add_node(agent_3)

builder.add_edge(START, "agent_1")
network = builder.compile()
```

### 감독자 (Supervisor)

이 아키텍처에서는 에이전트를 노드로 정의하고 다음에 호출해야 할 에이전트 노드를 결정하는 감독자 노드(LLM)를 추가합니다. [`Command`](./low_level.md#command)를 사용하여 감독자의 결정에 따라 실행을 적절한 에이전트 노드로 라우팅합니다. 이 아키텍처는 <mark>여러 에이전트를 병렬로 실행하거나 [맵-리듀스(map-reduce)](../how-tos/map-reduce.ipynb) 패턴을 사용하는 데에도 적합</mark>합니다.

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def supervisor(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which agent to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_agent" field)
    response = model.invoke(...)
    # route to one of the agents or exit based on the supervisor's decision
    # if the supervisor returns "__end__", the graph will finish execution
    return Command(goto=response["next_agent"])

def agent_1(state: MessagesState) -> Command[Literal["supervisor"]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # and add any additional logic (different models, custom prompts, structured output, etc.)
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

def agent_2(state: MessagesState) -> Command[Literal["supervisor"]]:
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

builder = StateGraph(MessagesState)
builder.add_node(supervisor)
builder.add_node(agent_1)
builder.add_node(agent_2)

builder.add_edge(START, "supervisor")

supervisor = builder.compile()
```

감독자 멀티 에이전트 아키텍처의 예는 이 [튜토리얼](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/)을 확인하십시오.

### 감독자 (도구 호출) (Supervisor (tool-calling))

[감독자](#감독자-supervisor)

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import InjectedState, create_react_agent

model = ChatOpenAI()

# this is the agent function that will be called as tool
# notice that you can pass the state to the tool via InjectedState annotation
def agent_1(state: Annotated[dict, InjectedState]):
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # and add any additional logic (different models, custom prompts, structured output, etc.)
    response = model.invoke(...)
    # return the LLM response as a string (expected tool response format)
    # this will be automatically turned to ToolMessage
    # by the prebuilt create_react_agent (supervisor)
    return response.content

def agent_2(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

tools = [agent_1, agent_2]
# the simplest way to build a supervisor w/ tool-calling is to use prebuilt ReAct agent graph
# that consists of a tool-calling LLM node (i.e. supervisor) and a tool-executing node
supervisor = create_react_agent(model, tools)
```

### 계층적 (Hierarchical)

시스템에 더 많은 에이전트를 추가하면 감독자가 모든 에이전트를 관리하기가 너무 어려워질 수 있습니다. 감독자가 다음에 호출할 에이전트에 대한 결정을 제대로 내리지 못하거나, 컨텍스트가 너무 복잡해져서 단일 감독자가 추적하기 어려워질 수 있습니다. 즉, 애초에 멀티 에이전트 아키텍처를 사용하게 된 동기와 동일한 문제에 직면하게 됩니다.

이를 해결하기 위해 시스템을 _계층적(hierarchically)_ 으로 설계할 수 있습니다. 예를 들어 개별 감독자가 관리하는 별도의 전문 에이전트 팀과 팀을 관리하는 최상위 감독자를 만들 수 있습니다.

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command
model = ChatOpenAI()

# define team 1 (same as the single supervisor example above)

def team_1_supervisor(state: MessagesState) -> Command[Literal["team_1_agent_1", "team_1_agent_2", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_agent"])

def team_1_agent_1(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

def team_1_agent_2(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

team_1_builder = StateGraph(Team1State)
team_1_builder.add_node(team_1_supervisor)
team_1_builder.add_node(team_1_agent_1)
team_1_builder.add_node(team_1_agent_2)
team_1_builder.add_edge(START, "team_1_supervisor")
team_1_graph = team_1_builder.compile()

# define team 2 (same as the single supervisor example above)
class Team2State(MessagesState):
    next: Literal["team_2_agent_1", "team_2_agent_2", "__end__"]

def team_2_supervisor(state: Team2State):
    ...

def team_2_agent_1(state: Team2State):
    ...

def team_2_agent_2(state: Team2State):
    ...

team_2_builder = StateGraph(Team2State)
...
team_2_graph = team_2_builder.compile()


# define top-level supervisor

builder = StateGraph(MessagesState)
def top_level_supervisor(state: MessagesState) -> Command[Literal["team_1_graph", "team_2_graph", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which team to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_team" field)
    response = model.invoke(...)
    # route to one of the teams or exit based on the supervisor's decision
    # if the supervisor returns "__end__", the graph will finish execution
    return Command(goto=response["next_team"])

builder = StateGraph(MessagesState)
builder.add_node(top_level_supervisor)
builder.add_node("team_1_graph", team_1_graph)
builder.add_node("team_2_graph", team_2_graph)
builder.add_edge(START, "top_level_supervisor")
builder.add_edge("team_1_graph", "top_level_supervisor")
builder.add_edge("team_2_graph", "top_level_supervisor")
graph = builder.compile()
```

### 사용자 정의 멀티 에이전트 워크플로 (Custom multi-agent workflow)

이 아키텍처에서는 개별 에이전트를 그래프 노드(graph nodes)로 추가하고 에이전트가 호출되는 순서를 사용자 정의 워크플로에서 미리 정의합니다. LangGraph에서 워크플로는 두 가지 방식으로 정의할 수 있습니다.

- **명시적 제어 흐름 (일반 엣지)(Explicit control flow (normal edges))**: <mark>LangGraph를 사용하면 애플리케이션의 제어 흐름 (즉, 에이전트가 통신하는 순서)을 [일반 그래프 엣지(normal graph edges)](./low_level.md#normal-edges)를 통해 명시적으로 정의</mark>할 수 있습니다. 이는 위 아키텍처의 가장 결정적인 변형입니다. <mark>다음에 어떤 에이전트가 호출될지 항상 미리 알 수 있습니다.</mark>

- **동적 제어 흐름 (Command)(Dynamic control flow (Command))**: <mark>LangGraph에서는 LLM이 애플리케이션 제어 흐름의 일부를 결정하도록 허용할 수 있습니다.</mark> 이는 [`Command`](./low_level.md#command)를 사용하여 달성할 수 있습니다. 이의 특별한 경우는 [감독자 도구 호출(supervisor tool-calling)](#supervisor-tool-calling) 아키텍처입니다. 이 경우 감독자 에이전트에 권한을 제안하는 도구 호출 LLM은 도구(에이전트)가 호출되는 순서에 대한 결정을 내립니다.

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START

model = ChatOpenAI()

def agent_1(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

def agent_2(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
# define the flow explicitly
builder.add_edge(START, "agent_1")
builder.add_edge("agent_1", "agent_2")
```

## 에이전트 간 통신 (Communication between agents)

멀티 에이전트 시스템을 구축할 때 가장 중요한 것은 에이전트가 어떻게 통신하는지 파악하는 것입니다. 몇 가지 고려 사항이 있습니다.

[**그래프 상태(graph state)를 통해 통신하는가, 아니면 도구 호출(tool calls)을 통해 통신하는가**](#그래프-상태-vs-도구-호출-graph-state-vs-tool-calls)
[**서로 다른 상태 스키마(state schemas)를 가지고 있는 경우**](#서로-다른-상태-스키마-different-state-schemas)
[**공유 메시지 목록(shared message list)**](#공유-메시지-목록-shared-message-list)

### 그래프 상태 vs 도구 호출 (Graph state vs tool calls)

에이전트 간에 전달되는 "페이로드(payload)"는 무엇입니까? 위에서 설명한 대부분의 아키텍처에서 에이전트는 [그래프 상태(graph state)](./low_level.md#state)를 통해 통신합니다. [도구 호출을 사용하는 감독자(supervisor with tool-calling)](#supervisor-tool-calling)의 경우 페이로드는 도구 호출 인수입니다.

![](./img/multi_agent/request.png)

#### 그래프 상태 (Graph state)

그래프 상태(graph state)를 통해 통신하려면 개별 에이전트를 [그래프 노드(graph nodes)](./low_level.md#nodes)로 정의해야 합니다. 이는 함수 또는 전체 [서브 그래프(subgraphs)](./low_level.md#subgraphs)로 추가할 수 있습니다. 그래프 실행의 각 단계에서 에이전트 노드는 그래프의 현재 상태를 수신하고 에이전트 코드를 실행한 다음 업데이트된 상태를 다음 노드로 전달합니다.

일반적으로 에이전트 노드는 단일 [상태 스키마(state schema)](./low_level.md#schema)를 공유합니다. 그러나 [서로 다른 상태 스키마(state schemas)](#different-state-schemas)를 가진 에이전트 노드를 설계할 수도 있습니다.

### 서로 다른 상태 스키마 (Different state schemas)

에이전트는 나머지 에이전트와 다른 상태 스키마(state schema)를 가져야 할 수 있습니다. 예를 들어 검색 에이전트는 쿼리 및 검색된 문서만 추적하면 될 수 있습니다. LangGraph에서 이를 달성하는 방법에는 두 가지가 있습니다.

- 별도의 상태 스키마(state schema)를 사용하여 [서브 그래프(subgraph)](./low_level.md#subgraphs) 에이전트를 정의합니다. 서브 그래프와 상위 그래프 간에 공유 상태 키(채널)가 없는 경우, 상위 그래프가 서브 그래프와 통신하는 방법을 알 수 있도록 [입력/출력 변환](https://langchain-ai.github.io/langgraph/how-tos/subgraph-transform-state/)을 추가하는 것이 중요합니다.
- 전체 그래프 상태 스키마와 구별되는 [개인 입력 상태 스키마(private input state schema)](https://langchain-ai.github.io/langgraph/how-tos/pass_private_state/)를 사용하여 에이전트 노드 함수를 정의합니다. 이를 통해 해당 특정 에이전트를 실행하는 데만 필요한 정보를 전달할 수 있습니다.

### 공유 메시지 목록 (Shared message list)

[전체 기록을 공유(share full history)](#전체-기록-공유-share-full-history)

![](./img/multi_agent/response.png)

#### 전체 기록 공유 (Share full history)

에이전트는 사고 과정의 **전체 기록(full history)** (즉, "스크래치패드(scratchpad)")를 다른 모든 에이전트와 **공유(share)**할 수 있습니다. 이 "스크래치패드(scratchpad)"는 일반적으로 [메시지 목록(list of messages)](./low_level.md#why-use-messages)처럼 보입니다. 전체 사고 과정을 공유하는 이점은 다른 에이전트가 더 나은 결정을 내리고 시스템 전체의 추론 능력을 향상시키는 데 도움이 될 수 있다는 것입니다. 단점은 에이전트 수와 복잡성이 증가함에 따라 "스크래치패드(scratchpad)"가 빠르게 증가하고 [메모리 관리(memory management)](./memory.md/#managing-long-conversation-history)를 위한 추가 전략이 필요할 수 있다는 것입니다.

#### 최종 결과 공유 (Share final result)

[서로 다른 상태 스키마(state schemas)](#서로-다른-상태-스키마-different-state-schemas)

도구로 호출되는 에이전트의 경우 감독자는 도구 스키마(tool schema)를 기반으로 입력을 결정합니다. 또한 LangGraph는 런타임 시 개별 도구에 [상태 전달(passing state)](https://langchain-ai.github.io/langgraph/how-tos/pass-run-time-values-to-tools/#pass-graph-state-to-tools)을 허용하므로 필요한 경우 하위 에이전트가 상위 상태에 액세스할 수 있습니다.
