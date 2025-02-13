# LangGraph 용어집

## 그래프 (Graphs)

LangGraph는 에이전트 워크플로우를 그래프로 모델링합니다. 에이전트의 동작은 다음 세 가지 핵심 구성 요소를 사용하여 정의합니다.

1. [`State`](#state) (상태): 애플리케이션의 현재 스냅샷을 나타내는 공유 데이터 구조입니다. 이는 모든 Python Type이 될 수 있지만, 일반적으로 `TypedDict` 또는 Pydantic `BaseModel`입니다.
2. [`Nodes`](#nodes) (노드): 에이전트의 로직을 인코딩하는 Python 함수입니다. 현재 `State`를 입력으로 받아 일부 계산 또는 외부 작업(side-effects)을 수행하고 업데이트된 `State`를 반환합니다.
3. [`Edges`](#edges) (에지): 현재 `State`를 기반으로 다음에 실행할 `Node`를 결정하는 Python 함수입니다. 조건부 분기 또는 고정 전환이 될 수 있습니다.

`Nodes`와 `Edges`를 구성하여 시간이 지남에 따라 `State`를 발전시키는 복잡한 루핑 워크플로우를 만들 수 있습니다. 그러나 <mark>진정한 힘은 LangGraph가 해당 `State`를 관리하는 방식에서 비롯됩니다.</mark> 강조하자면 `Nodes`와 `Edges`는 단순한 Python 함수일 뿐이며, LLM 또는 일반적인 Python 코드를 포함할 수 있습니다.

요약하자면: _노드는 작업을 수행하고, 에지는 다음에 무엇을 할지 알려줍니다._

LangGraph의 기본 그래프 알고리즘은 [메시지 전달](https://en.wikipedia.org/wiki/Message_passing)을 사용하여 일반적인 프로그램을 정의합니다. Node가 작업을 완료하면 하나 이상의 에지를 따라 다른 노드로 메시지를 보냅니다. 이러한 수신 노드는 해당 함수를 실행하고 결과 메시지를 다음 노드 집합으로 전달하며 프로세스가 계속됩니다. Google의 [Pregel](https://research.google.com/pubs/pregel-a-system-for-large-scale-graph-processing/) 시스템에서 영감을 받아 프로그램은 개별적인 "슈퍼-스텝(super-steps)"으로 진행됩니다.

슈퍼-스텝은 그래프 노드에 대한 단일 반복으로 간주할 수 있습니다. 병렬로 실행되는 노드는 동일한 슈퍼-스텝의 일부이고, 순차적으로 실행되는 노드는 별도의 슈퍼-스텝에 속합니다. 그래프 실행이 시작될 때 모든 노드는 `inactive` (비활성) 상태로 시작합니다. 노드는 들어오는 에지("채널(channels)")에서 새로운 메시지(상태)를 수신할 때 `active` (활성) 상태가 됩니다. 활성 노드는 해당 함수를 실행하고 업데이트로 응답합니다. 각 슈퍼-스텝이 끝나면 들어오는 메시지가 없는 노드는 스스로를 `inactive`로 표시하여 `halt` (중단)하도록 투표합니다. 모든 노드가 `inactive`이고 전송 중인 메시지가 없으면 그래프 실행이 종료됩니다.

### StateGraph

`StateGraph` 클래스는 사용할 주요 그래프 클래스입니다. 이는 사용자 정의 `State` 객체에 의해 매개변수화됩니다.

### 그래프 컴파일 (Compiling your graph)

그래프를 빌드하려면 먼저 [상태(state)](#state)를 정의한 다음 [노드(nodes)](#nodes)와 [에지(edges)](#edges)를 추가한 다음 컴파일합니다. 그래프 컴파일은 정확히 무엇이며 왜 필요할까요?

컴파일(Compiling)은 매우 간단한 단계입니다. 그래프 구조에 대한 몇 가지 기본적인 검사(고아 노드(orphaned nodes) 유무 등)를 제공합니다. 또한 [체크포인트(checkpointers)](./persistence.md) 및 [중단점(breakpoints)](#breakpoints)과 같은 런타임 인수(runtime args)를 지정할 수 있는 곳이기도 합니다. `.compile` 메서드를 호출하여 그래프를 컴파일합니다:

```python
graph = graph_builder.compile(...)
```

그래프를 사용하기 전에 **반드시** 컴파일해야 합니다.

## State (상태)

그래프를 정의할 때 가장 먼저 해야 할 일은 그래프의 `State` (상태)를 정의하는 것입니다. `State`는 [그래프의 스키마(schema of the graph)](#schema)와 상태 업데이트를 적용하는 방법을 지정하는 [`reducer` 함수](#reducers)로 구성됩니다. `State`의 스키마는 그래프의 모든 `Nodes` (노드)와 `Edges` (에지)에 대한 입력 스키마가 되며, `TypedDict` 또는 `Pydantic` 모델일 수 있습니다. 모든 `Nodes`는 `State`에 대한 업데이트를 내보내고, 지정된 `reducer` 함수를 사용하여 적용됩니다.

### Schema (스키마)

그래프의 스키마를 지정하는 주요 문서화된 방법은 `TypedDict`를 사용하는 것입니다. 그러나 **기본값(default values)** 및 추가 데이터 유효성 검사(data validation)를 추가하기 위해 [Pydantic BaseModel 사용](../how-tos/state-model.ipynb)을 그래프 상태로 지원합니다.

기본적으로 그래프는 동일한 입력 및 출력 스키마를 갖습니다. 이를 변경하려면 명시적인 입력 및 출력 스키마를 직접 지정할 수도 있습니다. 이는 키가 많고 일부는 명시적으로 입력용이고 다른 일부는 출력용인 경우에 유용합니다. 사용 방법은 [여기 노트북](../how-tos/input_output_schema.ipynb)을 참조하십시오.

#### Multiple schemas (다중 스키마)

일반적으로 모든 그래프 노드는 단일 스키마와 통신합니다. 즉, 동일한 상태 채널(state channels)을 읽고 씁니다. 그러나 이에 대한 더 많은 제어가 필요한 경우가 있습니다.

- 내부 노드는 그래프의 입력/출력에 필요하지 않은 정보를 전달할 수 있습니다.
- 그래프에 대해 서로 다른 입력/출력 스키마를 사용할 수도 있습니다. 예를 들어 출력에는 단일 관련 출력 키만 포함될 수 있습니다.

내부 노드 통신을 위해 노드가 그래프 내부의 개인 상태 채널(private state channels)에 쓸 수 있습니다. 간단히 개인 스키마인 `PrivateState`를 정의할 수 있습니다. 자세한 내용은 [이 노트북](../how-tos/pass_private_state.ipynb)을 참조하십시오.

그래프에 대한 명시적인 입력 및 출력 스키마를 정의할 수도 있습니다. 이러한 경우 그래프 작업과 관련된 _모든_ 키를 포함하는 "내부(internal)" 스키마를 정의합니다. 그러나 그래프의 입력 및 출력을 제한하기 위해 "내부" 스키마의 하위 집합인 `input` 및 `output` 스키마도 정의합니다. 자세한 내용은 [이 노트북](../how-tos/input_output_schema.ipynb)을 참조하십시오.

예제를 살펴 보겠습니다.

```python
class InputState(TypedDict):
    user_input: str

class OutputState(TypedDict):
    graph_output: str

class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str

class PrivateState(TypedDict):
    bar: str

def node_1(state: InputState) -> OverallState:
    # Write to OverallState
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    # Read from OverallState, write to PrivateState
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    # Read from PrivateState, write to OutputState
    return {"graph_output": state["bar"] + " Lance"}

builder = StateGraph(OverallState,input=InputState,output=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
graph.invoke({"user_input":"My"})
{'graph_output': 'My name is Lance'}
```

여기서 주목해야 할 두 가지 미묘하고 중요한 사항이 있습니다.

1. `state: InputState`를 `node_1`에 대한 입력 스키마로 전달합니다. 그러나 `OverallState`의 채널인 `foo`에 씁니다. 입력 스키마에 포함되지 않은 상태 채널에 어떻게 쓸 수 있을까요? 이는 노드가 _그래프 상태의 모든 상태 채널에 쓸 수 있기 때문입니다_. 그래프 상태는 초기화 시 정의된 상태 채널의 합집합이며, 여기에는 `OverallState`와 필터 `InputState` 및 `OutputState`가 포함됩니다.
2. `StateGraph(OverallState,input=InputState,output=OutputState)`로 그래프를 초기화합니다. 그렇다면 `node_2`에서 `PrivateState`에 어떻게 쓸 수 있을까요? `StateGraph` 초기화에 전달되지 않은 경우 그래프가 이 스키마에 어떻게 액세스할 수 있을까요? 노드는 상태 스키마 정의가 존재하는 한 _추가 상태 채널을 선언할 수도 있기 때문입니다_. 이 경우 `PrivateState` 스키마가 정의되어 있으므로 `bar`를 그래프에 새 상태 채널로 추가하고 쓸 수 있습니다.

### Reducers (리듀서)

리듀서(Reducers)는 노드의 업데이트가 `State` (상태)에 적용되는 방식을 이해하는 데 중요합니다. `State` (상태)의 각 키에는 자체 독립적인 리듀서(reducer) 함수가 있습니다. 명시적으로 지정된 리듀서(reducer) 함수가 없으면 해당 키에 대한 모든 업데이트가 재정의되는 것으로 간주됩니다. 기본 유형의 리듀서(reducer)부터 시작하여 몇 가지 다른 유형의 리듀서(reducer)가 있습니다.

#### Default Reducer (기본 리듀서)

다음 두 예제는 기본 리듀서(reducer)를 사용하는 방법을 보여줍니다.

**Example A:**

```python
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

이 예제에서는 어떤 키(key)에 대해서도 리듀서(reducer) 함수가 지정되지 않았습니다. 그래프에 대한 입력이 `{"foo": 1, "bar": ["hi"]}`라고 가정해 보겠습니다. 그런 다음 첫 번째 `Node`가 `{"foo": 2}`를 반환한다고 가정합니다. 이는 상태(state)에 대한 업데이트로 처리됩니다. `Node`가 전체 `State` (상태) 스키마를 반환할 필요는 없고 업데이트만 반환하면 된다는 점에 유의하세요. 이 업데이트를 적용한 후 `State` (상태)는 `{"foo": 2, "bar": ["hi"]}`가 됩니다. 두 번째 노드(node)가 `{"bar": ["bye"]}`를 반환하면 `State` (상태)는 `{"foo": 2, "bar": ["bye"]}`가 됩니다.

**예제 B:**

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

이 예제에서는 `Annotated` 유형을 사용하여 두 번째 키(`bar`)에 대한 리듀서 함수(`operator.add`)를 지정했습니다. 첫 번째 키는 변경되지 않은 상태로 유지됩니다. 그래프에 대한 입력이 `{"foo": 1, "bar": ["hi"]}`라고 가정해 보겠습니다. 그런 다음 첫 번째 `Node`가 `{"foo": 2}`를 반환한다고 가정합니다. 이는 상태(state)에 대한 업데이트로 처리됩니다. `Node`가 전체 `State` (상태) 스키마를 반환할 필요는 없고 업데이트만 반환하면 된다는 점에 유의하세요. 이 업데이트를 적용한 후 `State` (상태)는 `{"foo": 2, "bar": ["hi"]}`가 됩니다. 두 번째 노드(node)가 `{"bar": ["bye"]}`를 반환하면 `State` (상태)는 `{"foo": 2, "bar": ["hi", "bye"]}`가 됩니다. 여기에서 `bar` 키는 두 목록을 함께 추가하여 업데이트됩니다.

### Working with Messages in Graph State (그래프 상태에서 메시지 작업하기)

#### Why use messages? (왜 메시지를 사용해야 할까요?)

대부분의 최신 LLM (Large Language Model) 제공업체는 메시지 목록을 입력으로 허용하는 채팅 모델 인터페이스를 가지고 있습니다. 특히 LangChain의 [`ChatModel`](https://python.langchain.com/docs/concepts/#chat-models)은 `Message` 객체 목록을 입력으로 허용합니다. 이러한 메시지는 `HumanMessage` (사용자 입력) 또는 `AIMessage` (LLM 응답)와 같은 다양한 형태로 제공됩니다. 메시지 객체가 무엇인지 자세히 알아보려면 [이](https://python.langchain.com/docs/concepts/#messages) 개념 가이드를 참조하십시오.

#### Using Messages in your Graph (그래프에서 메시지 사용하기)

대부분의 경우 이전 대화 기록을 그래프 상태에 메시지 목록으로 저장하는 것이 유용합니다. 이렇게 하려면 그래프 상태에 `Message` 객체 목록을 저장하는 키(채널)를 추가하고 리듀서 함수로 주석을 달 수 있습니다 (아래 예제의 `messages` 키 참조). 리듀서 함수는 노드가 업데이트를 보낼 때마다 그래프가 상태의 `Message` 객체 목록을 업데이트하는 방법을 알려주는 데 중요합니다. 리듀서를 지정하지 않으면 모든 상태 업데이트는 메시지 목록을 가장 최근에 제공된 값으로 덮어씁니다. 기존 목록에 메시지를 간단히 추가하려면 `operator.add`를 리듀서로 사용할 수 있습니다.

그러나 그래프 상태에서 메시지를 수동으로 업데이트할 수도 있습니다 (예: human-in-the-loop). `operator.add`를 사용하는 경우 그래프로 보내는 수동 상태 업데이트가 기존 메시지를 업데이트하는 대신 기존 메시지 목록에 추가됩니다. 이를 방지하려면 메시지 ID를 추적하고 업데이트된 경우 기존 메시지를 덮어쓸 수 있는 리듀서가 필요합니다. 이를 위해 미리 빌드된 `add_messages` 함수를 사용할 수 있습니다. 새로운 메시지의 경우 기존 목록에 간단히 추가되지만 기존 메시지에 대한 업데이트도 올바르게 처리합니다.

#### Serialization (직렬화)

메시지 ID를 추적하는 것 외에도 `add_messages` 함수는 `messages` 채널에서 상태 업데이트가 수신될 때마다 메시지를 LangChain `Message` 객체로 역직렬화하려고 시도합니다. LangChain 직렬화/역직렬화에 대한 자세한 내용은 [여기](https://python.langchain.com/docs/how_to/serialization/)를 참조하십시오. 이를 통해 다음 형식으로 그래프 입력/상태 업데이트를 보낼 수 있습니다.

```python
# this is supported
{"messages": [HumanMessage(content="message")]}

# and this is also supported
{"messages": [{"type": "human", "content": "message"}]}
```

`add_messages`를 사용할 때 상태 업데이트(state updates)는 항상 LangChain의 `Messages`로 역직렬화(deserialized)되므로, `state["messages"][-1].content`와 같이 점 표기법(dot notation)을 사용하여 메시지 속성에 접근해야 합니다. 아래는 `add_messages`를 리듀서(reducer) 함수로 사용하는 그래프의 예시입니다.

```python
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

#### MessagesState

상태(state)에 메시지 목록을 포함하는 것이 매우 일반적이므로 메시지를 쉽게 사용할 수 있도록 미리 빌드된 상태(state)인 `MessagesState`가 있습니다. `MessagesState`는 `AnyMessage` 객체 목록인 단일 `messages` 키로 정의되며 `add_messages` 리듀서(reducer)를 사용합니다. 일반적으로 메시지 외에도 추적해야 할 상태(state)가 더 많으므로 사람들은 이 상태(state)를 서브클래스화하고 다음과 같이 더 많은 필드를 추가합니다.

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

## 노드 (Nodes)

LangGraph에서 노드(nodes)는 일반적으로 파이썬 함수(python functions) (동기 또는 비동기)이며, 여기서 **첫 번째** 위치 인수는 [상태(state)](#state)이고, (선택적으로) **두 번째** 위치 인수는 선택적인 [구성 가능한 매개변수(configurable parameters)](#configuration) (예: `thread_id`)를 포함하는 "config"입니다.

`NetworkX`와 유사하게, [add_node][langgraph.graph.StateGraph.add_node] 메서드를 사용하여 이러한 노드(nodes)를 그래프에 추가합니다:

```python
from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph

builder = StateGraph(dict)


def my_node(state: dict, config: RunnableConfig):
    print("In node: ", config["configurable"]["user_id"])
    return {"results": f"Hello, {state['input']}!"}


# The second argument is optional
def my_other_node(state: dict):
    return state


builder.add_node("my_node", my_node)
builder.add_node("other_node", my_other_node)
...
```

내부적으로 함수는 [RunnableLambda](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.base.RunnableLambda.html#langchain_core.runnables.base.RunnableLambda)로 변환됩니다. 이는 함수에 기본 추적(tracing) 및 디버깅(debugging)과 함께 배치(batch) 및 비동기(async) 지원을 추가합니다.

이름을 지정하지 않고 그래프에 노드(node)를 추가하면 함수 이름과 동일한 기본 이름이 지정됩니다.

```python
builder.add_node(my_node)
# You can then create edges to/from this node by referencing it as `"my_node"`
```

### `START` Node

### `START` 노드

`START` 노드는 그래프에 사용자 입력을 전송하는 특별한 노드입니다. 이 노드를 참조하는 주된 목적은 어떤 노드가 가장 먼저 호출되어야 하는지 결정하는 것입니다.

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### `END` 노드

`END` 노드는 종료 노드를 나타내는 특별한 노드입니다. 이 노드는 더 이상 수행할 작업이 없는 엣지(edge)를 표시하고자 할 때 참조됩니다.

```
from langgraph.graph import END

graph.add_edge("node_a", END)
```

## 엣지 (Edges)

엣지(Edges)는 로직이 어떻게 라우팅되고 그래프가 언제 멈출지 결정하는 방법을 정의합니다. 이는 에이전트가 작동하는 방식과 서로 다른 노드들이 통신하는 방식에서 매우 중요한 부분입니다. 엣지에는 몇 가지 주요 유형이 있습니다.

- 일반 엣지 (Normal Edges): 한 노드에서 다음 노드로 직접 이동합니다.
- 조건부 엣지 (Conditional Edges): 함수를 호출하여 다음에 이동할 노드를 결정합니다.
- 진입점 (Entry Point): 사용자 입력이 도착했을 때 가장 먼저 호출할 노드입니다.
- 조건부 진입점 (Conditional Entry Point): 함수를 호출하여 사용자 입력이 도착했을 때 가장 먼저 호출할 노드를 결정합니다.

하나의 노드는 여러 개의 나가는 엣지(outgoing edges)를 가질 수 있습니다. 노드에 여러 개의 나가는 엣지가 있는 경우, 이러한 모든 대상 노드는 다음 슈퍼스텝(superstep)의 일부로 병렬로 실행됩니다.

### 일반 엣지 (Normal Edges)

노드 A에서 노드 B로 **항상** 이동하려는 경우, [add_edge][langgraph.graph.StateGraph.add_edge] 메서드를 직접 사용할 수 있습니다.

```python
graph.add_edge("node_a", "node_b")
```

### 조건부 엣지 (Conditional Edges)

하나 이상의 엣지로 **선택적으로** 라우팅하거나 (또는 선택적으로 종료하려는 경우) [add_conditional_edges][langgraph.graph.StateGraph.add_conditional_edges] 메서드를 사용할 수 있습니다. 이 메서드는 노드 이름과 해당 노드가 실행된 후 호출할 "라우팅 함수(routing function)"의 이름을 인수로 받습니다.

```python
graph.add_conditional_edges("node_a", routing_function)
```

노드와 유사하게, `routing_function`은 그래프의 현재 `state` (상태)를 받아 값을 반환합니다.

기본적으로 `routing_function`의 반환 값은 상태를 다음에 보낼 노드 (또는 노드 목록)의 이름으로 사용됩니다. 이러한 모든 노드는 다음 슈퍼스텝(superstep)의 일부로 병렬로 실행됩니다.

선택적으로 `routing_function`의 출력을 다음 노드의 이름에 매핑하는 사전을 제공할 수 있습니다.

```python
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

!!! tip
[`Command`](#command-명령)

### 진입점 (Entry Point)

진입점(Entry point)은 그래프가 시작될 때 가장 먼저 실행되는 노드입니다. 가상 [`START`][langgraph.constants.START] 노드에서 실행할 첫 번째 노드로 [`add_edge`][langgraph.graph.StateGraph.add_edge] 메서드를 사용하여 그래프에 진입할 위치를 지정할 수 있습니다.

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### 조건부 진입점 (Conditional Entry Point)

조건부 진입점(Conditional entry point)을 사용하면 사용자 정의 로직에 따라 다른 노드에서 시작할 수 있습니다. 이를 위해 가상 [`START`][langgraph.constants.START] 노드에서 [`add_conditional_edges`][langgraph.graph.StateGraph.add_conditional_edges]를 사용할 수 있습니다.

```python
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)
```

선택적으로 `routing_function`의 출력을 다음 노드의 이름에 매핑하는 사전을 제공할 수 있습니다.

```python
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})
```

## `Send` (전송)

기본적으로 `Nodes`(노드)와 `Edges`(엣지)는 미리 정의되어 있고 동일한 공유 상태에서 작동합니다. 하지만 정확한 엣지를 미리 알 수 없거나 `State`(상태)의 다른 버전이 동시에 존재하기를 원하는 경우가 있을 수 있습니다. 이러한 예로는 `map-reduce` 디자인 패턴이 있습니다. 이 디자인 패턴에서는 첫 번째 노드가 객체 목록을 생성하고, 이러한 모든 객체에 다른 노드를 적용하고 싶을 수 있습니다. 객체의 수를 미리 알 수 없을 수 있으며(즉, 엣지의 수를 알 수 없음), 다운스트림 `Node`에 대한 입력 `State`는 서로 달라야 합니다(생성된 각 객체마다 하나씩).

이러한 디자인 패턴을 지원하기 위해 LangGraph는 조건부 엣지에서 [`Send`][langgraph.types.Send] 객체를 반환하는 것을 지원합니다. `Send`는 두 개의 인수를 받습니다: 첫 번째는 노드의 이름이고, 두 번째는 해당 노드에 전달할 상태입니다.

```python
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)
```

## `Command` (명령)

제어 흐름(edges)과 상태 업데이트(nodes)를 결합하는 것이 유용할 수 있습니다. 예를 들어, 동일한 노드에서 상태 업데이트를 수행하면서 동시에 다음에 이동할 노드를 결정하고 싶을 수 있습니다. LangGraph는 노드 함수에서 [`Command`][langgraph.types.Command] 객체를 반환하여 이를 수행할 수 있는 방법을 제공합니다:

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```

With `Command` you can also achieve dynamic control flow behavior (identical to [conditional edges](#conditional-edges)):

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")
```

!!! important

```
노드 함수에서 `Command`를 반환할 때는 노드가 라우팅하는 노드 이름 목록을 포함하는 반환 타입 어노테이션을 추가해야 합니다. 예: `Command[Literal["my_other_node"]]`. 이는 그래프 렌더링에 필요하며 `my_node`가 `my_other_node`로 이동할 수 있다는 것을 LangGraph에 알려줍니다.
```

`Command` 사용 방법에 대한 전체 예제는 이 [사용 방법 가이드](../how-tos/command.ipynb)를 확인하세요.

### 조건부 엣지 대신 Command를 사용해야 하는 경우는 언제인가요?

그래프 상태를 업데이트하고 **동시에** 다른 노드로 라우팅해야 할 때 `Command`를 사용하세요. 예를 들어, 다른 에이전트에게 정보를 전달하고 라우팅해야 하는 [멀티 에이전트 핸드오프](./multi_agent.md#handoffs)를 구현할 때 사용합니다.

상태를 업데이트하지 않고 노드 간에 조건부로 라우팅하려면 [조건부 엣지](#conditional-edges)를 사용하세요.

### 부모 그래프의 노드로 이동하기

[서브그래프](#서브그래프-subgraphs)

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # where `other_subgraph` is a node in the parent graph
        graph=Command.PARENT
    )
```

!!! note

```
`graph`를 `Command.PARENT`로 설정하면 가장 가까운 부모 그래프로 이동합니다.
```

!!! important "Command.PARENT와 함께 상태 업데이트하기"

```
부모와 서브그래프 [상태 스키마](#schema)가 공유하는 키에 대해 서브그래프 노드에서 부모 그래프 노드로 업데이트를 보낼 때는, 부모 그래프 상태에서 업데이트하는 키에 대한 [리듀서](#reducers)를 정의해야 합니다. 이 [예제](../how-tos/command.ipynb#navigating-to-a-node-in-a-parent-graph)를 참조하세요.
```

이는 특히 [멀티 에이전트 핸드오프](./multi_agent.md#handoffs)를 구현할 때 유용합니다.

### 도구 내에서 사용하기

일반적인 사용 사례는 도구 내에서 그래프 상태를 업데이트하는 것입니다. 예를 들어, 고객 지원 애플리케이션에서 대화 시작 시 계정 번호나 ID를 기반으로 고객 정보를 조회하고 싶을 수 있습니다. 도구에서 그래프 상태를 업데이트하려면 도구에서 `Command(update={"my_custom_key": "foo", "messages": [...]})` 를 반환할 수 있습니다:

```python
@tool
def lookup_user_info(tool_call_id: Annotated[str, InjectedToolCallId], config: RunnableConfig):
    """Use this to look up user information to better assist them with their questions."""
    user_info = get_user_info(config.get("configurable", {}).get("user_id"))
    return Command(
        update={
            # update the state keys
            "user_info": user_info,
            # update the message history
            "messages": [ToolMessage("Successfully looked up user information", tool_call_id=tool_call_id)]
        }
    )
```

!!! important
도구에서 `Command`를 반환할 때는 `Command.update`에 `messages`(또는 메시지 기록에 사용되는 모든 상태 키)를 포함해야 하며, `messages`의 메시지 목록에는 `ToolMessage`가 포함되어야 합니다. 이는 결과 메시지 기록이 유효하도록 하기 위해 필요합니다(LLM 제공자는 도구 호출이 있는 AI 메시지 다음에 도구 결과 메시지가 있어야 합니다).

`Command`를 통해 상태를 업데이트하는 도구를 사용하는 경우, `Command` 객체를 자동으로 처리하고 그래프 상태로 전파하는 사전 구축된 [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode]를 사용하는 것이 좋습니다. 도구를 호출하는 사용자 정의 노드를 작성하는 경우, 도구가 반환하는 `Command` 객체를 노드의 업데이트로 수동으로 전파해야 합니다.

### 휴먼 인 더 루프 (Human-in-the-loop)

`Command`는 휴먼 인 더 루프 워크플로우의 중요한 부분입니다: `interrupt()`를 사용하여 사용자 입력을 수집할 때, `Command`는 `Command(resume="User input")`를 통해 입력을 제공하고 실행을 재개하는 데 사용됩니다. 자세한 내용은 [이 개념 가이드](./human_in_the_loop.md)를 확인하세요.

## 지속성 (Persistence)

LangGraph는 [체크포인터][langgraph.checkpoint.base.BaseCheckpointSaver]를 사용하여 에이전트의 상태를 지속적으로 저장하는 기능을 제공합니다. 체크포인터는 모든 슈퍼스텝에서 그래프 상태의 스냅샷을 저장하여 언제든지 재개할 수 있도록 합니다. 이를 통해 휴먼 인 더 루프 상호작용, 메모리 관리, 장애 허용성과 같은 기능이 가능해집니다. 적절한 `get`과 `update` 메서드를 사용하여 그래프 실행 후에도 그래프의 상태를 직접 조작할 수 있습니다. 자세한 내용은 [지속성 개념 가이드](./persistence.md)를 참조하세요.

## 스레드 (Threads)

LangGraph의 스레드는 그래프와 사용자 간의 개별 세션이나 대화를 나타냅니다. 체크포인팅을 사용할 때, 단일 대화의 턴(그리고 단일 그래프 실행 내의 단계)은 고유한 스레드 ID로 구성됩니다.

## 저장소 (Storage)

LangGraph는 [BaseStore][langgraph.store.base.BaseStore] 인터페이스를 통해 내장 문서 저장소를 제공합니다. 스레드 ID로 상태를 저장하는 체크포인터와 달리, 저장소는 데이터를 구성하기 위해 사용자 정의 네임스페이스를 사용합니다. 이를 통해 스레드 간 지속성이 가능해져 에이전트가 장기 메모리를 유지하고, 과거 상호작용으로부터 학습하며, 시간이 지남에 따라 지식을 축적할 수 있습니다. 일반적인 사용 사례로는 사용자 프로필 저장, 지식 베이스 구축, 모든 스레드에서 전역 환경 설정 관리 등이 있습니다.

## 그래프 마이그레이션 (Graph Migrations)

LangGraph는 체크포인터를 사용하여 상태를 추적하는 경우에도 그래프 정의(노드, 엣지, 상태)의 마이그레이션을 쉽게 처리할 수 있습니다.

- 그래프 끝에 있는 스레드(즉, 중단되지 않은)의 경우 그래프의 전체 토폴로지를 변경할 수 있습니다(즉, 모든 노드와 엣지를 제거, 추가, 이름 변경 등)
- 현재 중단된 스레드의 경우, 노드 이름 변경/제거를 제외한 모든 토폴로지 변경을 지원합니다(해당 스레드가 더 이상 존재하지 않는 노드로 진입하려 할 수 있기 때문입니다) -- 이것이 문제가 된다면 연락 주시면 해결책을 우선순위로 검토하겠습니다.
- 상태 수정의 경우, 키 추가 및 제거에 대해 완전한 전/후방 호환성을 제공합니다
- 이름이 변경된 상태 키는 기존 스레드에서 저장된 상태를 잃게 됩니다
- 유형이 호환되지 않는 방식으로 변경된 상태 키는 현재 변경 전의 상태를 가진 스레드에서 문제를 일으킬 수 있습니다 -- 이것이 문제가 된다면 연락 주시면 해결책을 우선순위로 검토하겠습니다.

## 구성 (Configuration)

그래프를 생성할 때, 그래프의 특정 부분을 구성 가능하도록 표시할 수 있습니다. 이는 일반적으로 모델이나 시스템 프롬프트를 쉽게 전환할 수 있도록 하는 데 사용됩니다. 이를 통해 단일 "인지 아키텍처"(그래프)를 만들고 여러 다른 인스턴스를 가질 수 있습니다.

그래프를 생성할 때 선택적으로 `config_schema`를 지정할 수 있습니다.

```python
class ConfigSchema(TypedDict):
    llm: str

graph = StateGraph(State, config_schema=ConfigSchema)
```

그런 다음 `configurable` 구성 필드를 사용하여 이 구성을 그래프에 전달할 수 있습니다.

```python
config = {"configurable": {"llm": "anthropic"}}

graph.invoke(inputs, config=config)
```

그리고 노드 내에서 이 구성에 접근하고 사용할 수 있습니다:

```python
def node_a(state, config):
    llm_type = config.get("configurable", {}).get("llm", "openai")
    llm = get_llm(llm_type)
    ...
```

구성에 대한 전체 설명은 [이 가이드](../how-tos/configuration.ipynb)를 참조하세요.

### 재귀 제한 (Recursion Limit)

[슈퍼스텝](#그래프-graphs)

```python
graph.invoke(inputs, config={"recursion_limit": 5, "configurable":{"llm": "anthropic"}})
```

재귀 제한이 작동하는 방식에 대해 자세히 알아보려면 [이 사용 방법](https://langchain-ai.github.io/langgraph/how-tos/recursion-limit/)을 읽어보세요.

## `interrupt` (중단)

[interrupt](../reference/types.md/#langgraph.types.interrupt) 함수를 사용하여 특정 지점에서 그래프를 **일시 중지**하고 사용자 입력을 수집합니다. `interrupt` 함수는 중단 정보를 클라이언트에 표시하여 개발자가 사용자 입력을 수집하고, 그래프 상태를 검증하거나, 실행을 재개하기 전에 결정을 내릴 수 있도록 합니다.

```python
from langgraph.types import interrupt

def human_approval_node(state: State):
    ...
    answer = interrupt(
        # This value will be sent to the client.
        # It can be any JSON serializable value.
        {"question": "is it ok to continue?"},
    )
    ...
```

[`Command`](#command-명령)

**휴먼 인 더 루프** 워크플로우에서 `interrupt`가 어떻게 사용되는지에 대해서는 [휴먼 인 더 루프 개념 가이드](./human_in_the_loop.md)에서 자세히 알아보세요.

## 중단점 (Breakpoints)

중단점은 특정 지점에서 그래프 실행을 일시 중지하고 실행을 단계별로 진행할 수 있게 합니다. 중단점은 각 그래프 단계 후에 상태를 저장하는 LangGraph의 [**지속성 계층**](./persistence.md)에 의해 구동됩니다. 중단점은 [**휴먼 인 더 루프**](./human_in_the_loop.md) 워크플로우를 활성화하는 데도 사용할 수 있지만, 이 목적으로는 [`interrupt` 함수](#interrupt-function)를 사용하는 것이 좋습니다.

중단점에 대해 자세히 알아보려면 [중단점 개념 가이드](./breakpoints.md)를 참조하세요.

## 서브그래프 (Subgraphs)

[그래프](#그래프-graphs)

- [멀티 에이전트 시스템](./multi_agent.md) 구축하기
- 여러 그래프에서 일부 상태를 공유할 수 있는 노드 집합을 재사용하려는 경우, 서브그래프에서 한 번 정의하고 여러 부모 그래프에서 사용할 수 있습니다
- 서로 다른 팀이 그래프의 서로 다른 부분을 독립적으로 작업하기를 원할 때, 각 부분을 서브그래프로 정의할 수 있으며, 서브그래프 인터페이스(입력 및 출력 스키마)가 준수되는 한 부모 그래프는 서브그래프의 세부 사항을 알 필요 없이 구축될 수 있습니다

부모 그래프에 서브그래프를 추가하는 방법에는 두 가지가 있습니다:

- 컴파일된 서브그래프로 노드 추가: 부모 그래프와 서브그래프가 상태 키를 공유하고 입출력 시 상태를 변환할 필요가 없을 때 유용합니다

```python
builder.add_node("subgraph", subgraph_builder.compile())
```

- 서브그래프를 호출하는 함수로 노드 추가: 부모 그래프와 서브그래프가 서로 다른 상태 스키마를 가지고 있고 서브그래프를 호출하기 전이나 후에 상태를 변환해야 할 때 유용합니다

```python
subgraph = subgraph_builder.compile()

def call_subgraph(state: State):
    return subgraph.invoke({"subgraph_key": state["parent_key"]})

builder.add_node("subgraph", call_subgraph)
```

각각의 예를 살펴보겠습니다.

### 컴파일된 그래프로 사용

[컴파일된 서브그래프](#그래프-컴파일-compiling-your-graph)

!!! Note
서브그래프 노드에 추가 키를 전달하면(즉, 공유 키 외에) 서브그래프 노드에서 무시됩니다. 마찬가지로, 서브그래프에서 추가 키를 반환하면 부모 그래프에서 무시됩니다.

```python
from langgraph.graph import StateGraph
from typing import TypedDict

class State(TypedDict):
    foo: str

class SubgraphState(TypedDict):
    foo: str  # note that this key is shared with the parent graph state
    bar: str

# Define subgraph
def subgraph_node(state: SubgraphState):
    # note that this subgraph node can communicate with the parent graph via the shared "foo" key
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node)
...
subgraph = subgraph_builder.compile()

# Define parent graph
builder = StateGraph(State)
builder.add_node("subgraph", subgraph)
...
graph = builder.compile()
```

### 함수로 사용

완전히 다른 스키마를 가진 서브그래프를 정의하고 싶을 수 있습니다. 이 경우, 서브그래프를 호출하는 노드 함수를 만들 수 있습니다. 이 함수는 서브그래프를 호출하기 전에 입력(부모) 상태를 서브그래프 상태로 [변환](../how-tos/subgraph-transform-state.ipynb)하고, 결과를 부모 상태로 다시 변환한 후 노드에서 상태 업데이트를 반환해야 합니다.

```python
class State(TypedDict):
    foo: str

class SubgraphState(TypedDict):
    # note that none of these keys are shared with the parent graph state
    bar: str
    baz: str

# Define subgraph
def subgraph_node(state: SubgraphState):
    return {"bar": state["bar"] + "baz"}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node)
...
subgraph = subgraph_builder.compile()

# Define parent graph
def node(state: State):
    # transform the state to the subgraph state
    response = subgraph.invoke({"bar": state["foo"]})
    # transform response back to the parent state
    return {"foo": response["bar"]}

builder = StateGraph(State)
# note that we are using `node` function instead of a compiled subgraph
builder.add_node(node)
...
graph = builder.compile()
```

## 시각화 (Visualization)

그래프가 복잡해질수록 시각화하는 것이 유용할 수 있습니다. LangGraph에는 그래프를 시각화하는 여러 가지 내장 방법이 있습니다. 자세한 내용은 [이 사용 방법 가이드](../how-tos/visualization.ipynb)를 참조하세요.

## 스트리밍 (Streaming)

LangGraph는 그래프 노드의 실행 중 업데이트 스트리밍, LLM 호출의 토큰 스트리밍 등을 포함한 스트리밍을 기본적으로 지원하도록 구축되었습니다. 자세한 내용은 이 [개념 가이드](./streaming.md)를 참조하세요.

