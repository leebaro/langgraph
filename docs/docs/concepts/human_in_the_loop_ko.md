# Human-in-the-loop

!!! tip "This guide uses the new `interrupt` function."

    LangGraph 0.2.57부터는 **human-in-the-loop** 패턴을 단순화하므로 [`interrupt` 함수][langgraph.types.interrupt]를 사용하여 중단점(breakpoint)을 설정하는 것이 좋습니다.

    정적 중단점(static breakpoint)과 `NodeInterrupt` 예외에 의존했던 이전 버전의 컨셉 가이드(conceptual guide)는 [여기](v0-human-in-the-loop.md)에서 확인할 수 있습니다.

**Human-in-the-loop** (또는 "on-the-loop") 워크플로는 자동화된 프로세스에 사람의 입력을 통합하여 주요 단계에서 의사 결정, 유효성 검사 또는 수정이 가능하도록 합니다. 이는 기본 모델이 가끔 부정확한 결과를 생성할 수 있는 **LLM 기반 애플리케이션**에서 특히 유용합니다. 규정 준수, 의사 결정 또는 콘텐츠 생성과 같이 오류 허용 범위가 낮은 시나리오에서 사람의 참여는 모델 출력의 검토, 수정 또는 재정의를 통해 신뢰성을 보장합니다.

## Use cases (사용 사례)

LLM 기반 애플리케이션에서 **human-in-the-loop** 워크플로의 주요 사용 사례는 다음과 같습니다.

[**🛠️ Reviewing tool calls (도구 호출 검토)**](#review-tool-calls-도구-호출-검토)
2. **✅ Validating LLM outputs (LLM 출력 유효성 검사)**: LLM이 생성한 콘텐츠를 검토, 편집 또는 승인할 수 있습니다.
3. **💡 Providing context (컨텍스트 제공)**: LLM이 명확성 또는 추가 세부 정보를 위해 사람의 입력을 명시적으로 요청하거나 다중 턴 대화(multi-turn conversation)를 지원할 수 있도록 합니다.

## `interrupt`

LangGraph의 [`interrupt` 함수][langgraph.types.interrupt]는 특정 노드에서 그래프를 일시 중지하고, 사람에게 정보를 제공하고, 사람의 입력으로 그래프를 재개하여 human-in-the-loop 워크플로를 활성화합니다. 이 함수는 승인, 편집 또는 추가 입력 수집과 같은 작업에 유용합니다. [`interrupt` 함수][langgraph.types.interrupt]는 사람이 제공한 값으로 그래프를 재개하기 위해 [`Command`](../reference/types.md#langgraph.types.Command) 객체와 함께 사용됩니다.

```python
from langgraph.types import interrupt

def human_node(state: State):
    value = interrupt(
        # Any JSON serializable value to surface to the human.
        # For example, a question or a piece of text or a set of keys in the state
       {
          "text_to_revise": state["some_text"]
       }
    )
    # Update the state with the human's input or route the graph based on the input.
    return {
        "some_text": value
    }

graph = graph_builder.compile(
    checkpointer=checkpointer # Required for `interrupt` to work
)

# Run the graph until the interrupt
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(some_input, config=thread_config)
    
# Resume the graph with the human's input
graph.invoke(Command(resume=value_from_human), config=thread_config)
```

```pycon
{'some_text': 'Edited text'}
```

!!! warning
      Interrupts are both powerful and ergonomic. However, while they may resemble Python's input() function in terms of developer experience, it's important to note that they do not automatically resume execution from the interruption point. Instead, they rerun the entire node where the interrupt was used.
[resuming from an interrupt](#how-does-resuming-from-an-interrupt-work-interrupt에서-재개하는-방법)

??? "Full Code"

      Here's a full example of how to use `interrupt` in a graph, if you'd like
      to see the code in action.

      ```python
      from typing import TypedDict
      import uuid

      from langgraph.checkpoint.memory import MemorySaver
      from langgraph.constants import START
      from langgraph.graph import StateGraph
      from langgraph.types import interrupt, Command

      class State(TypedDict):
         """The graph state."""
         some_text: str

      def human_node(state: State):
         value = interrupt(
            # Any JSON serializable value to surface to the human.
            # For example, a question or a piece of text or a set of keys in the state
            {
               "text_to_revise": state["some_text"]
            }
         )
         return {
            # Update the state with the human's input
            "some_text": value
         }


      # Build the graph
      graph_builder = StateGraph(State)
      # Add the human-node to the graph
      graph_builder.add_node("human_node", human_node)
      graph_builder.add_edge(START, "human_node")

      # A checkpointer is required for `interrupt` to work.
      checkpointer = MemorySaver()
      graph = graph_builder.compile(
         checkpointer=checkpointer
      )

      # Pass a thread ID to the graph to run it.
      thread_config = {"configurable": {"thread_id": uuid.uuid4()}}

      # Using stream() to directly surface the `__interrupt__` information.
      for chunk in graph.stream({"some_text": "Original text"}, config=thread_config):
         print(chunk)

      # Resume using Command
      for chunk in graph.stream(Command(resume="Edited text"), config=thread_config):
         print(chunk)
      ```

      ```pycon
      {'__interrupt__': (
            Interrupt(
               value={'question': 'Please revise the text', 'some_text': 'Original text'}, 
               resumable=True, 
               ns=['human_node:10fe492f-3688-c8c6-0d0a-ec61a43fecd6'], 
               when='during'
            ),
         )
      }
      {'human_node': {'some_text': 'Edited text'}}
      ```

## Requirements (요구 사항)

그래프에서 `interrupt`를 사용하려면 다음이 필요합니다.

1. 각 단계 후에 그래프 상태를 저장하기 위해 [**Specify a checkpointer (체크포인터 지정)**](persistence.md#checkpoints)합니다.
[Design Patterns (디자인 패턴)](#design-patterns-디자인-패턴)
3. `interrupt`가 발생할 때까지 [**thread ID (스레드 ID)**](./persistence.md#threads)로 **Run the graph (그래프 실행)**합니다.
[**The `Command` primitive (`Command` 기본 요소)**](#the-command-primitive-command-기본-요소)

## Design Patterns (디자인 패턴)

일반적으로 human-in-the-loop 워크플로로 수행할 수 있는 세 가지 다른 **actions (작업)**이 있습니다.

1. **Approve or Reject (승인 또는 거부)**: API 호출과 같은 중요한 단계 전에 그래프를 일시 중지하여 작업을 검토하고 승인합니다. 작업이 거부되면 그래프가 단계를 실행하지 못하도록 하고 잠재적으로 대체 작업을 수행할 수 있습니다. 이 패턴은 종종 사람의 입력에 따라 그래프를 **routing (라우팅)**하는 것을 포함합니다.
2. **Edit Graph State (그래프 상태 편집)**: 그래프를 일시 중지하여 그래프 상태를 검토하고 편집합니다. 이는 실수를 수정하거나 추가 정보로 상태를 업데이트하는 데 유용합니다. 이 패턴은 종종 사람의 입력으로 상태를 **updating (업데이트)**하는 것을 포함합니다.
3. **Get Input (입력 받기)**: 그래프의 특정 단계에서 사람의 입력을 명시적으로 요청합니다. 이는 에이전트의 의사 결정 프로세스에 정보를 제공하거나 **multi-turn conversations (다중 턴 대화)**를 지원하기 위해 추가 정보 또는 컨텍스트를 수집하는 데 유용합니다.

아래에서는 이러한 **actions (작업)**을 사용하여 구현할 수 있는 다양한 디자인 패턴을 보여줍니다.

### Approve or Reject (승인 또는 거부)

<figure markdown="1">
![image](img/human_in_the_loop/approve-or-reject.png){: style="max-height:400px"}
<figcaption>Depending on the human's approval or rejection, the graph can proceed with the action or take an alternative path.</figcaption>
</figure>

API 호출과 같은 중요한 단계 전에 그래프를 일시 중지하여 작업을 검토하고 승인합니다. 작업이 거부되면 그래프가 단계를 실행하지 못하도록 하고 잠재적으로 대체 작업을 수행할 수 있습니다.

```python

from typing import Literal
from langgraph.types import interrupt, Command

def human_approval(state: State) -> Command[Literal["some_node", "another_node"]]:
    is_approved = interrupt(
        {
            "question": "Is this correct?",
            # Surface the output that should be
            # reviewed and approved by the human.
            "llm_output": state["llm_output"]
        }
    )

    if is_approved:
        return Command(goto="some_node")
    else:
        return Command(goto="another_node")

# Add the node to the graph in an appropriate location
# and connect it to the relevant nodes.
graph_builder.add_node("human_approval", human_approval)
graph = graph_builder.compile(checkpointer=checkpointer)

# After running the graph and hitting the interrupt, the graph will pause.
# Resume it with either an approval or rejection.
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(Command(resume=True), config=thread_config)
```

자세한 예제는 [how to review tool calls (도구 호출 검토 방법)](../how-tos/human_in_the_loop/review-tool-calls.ipynb)를 참조하십시오.

### Review & Edit State (상태 검토 및 편집)

<figure markdown="1">
![image](img/human_in_the_loop/edit-graph-state-simple.png){: style="max-height:400px"}
<figcaption>A human can review and edit the state of the graph. This is useful for correcting mistakes or updating the state with additional information.
</figcaption>
</figure>

```python
from langgraph.types import interrupt

def human_editing(state: State):
    ...
    result = interrupt(
        # Interrupt information to surface to the client.
        # Can be any JSON serializable value.
        {
            "task": "Review the output from the LLM and make any necessary edits.",
            "llm_generated_summary": state["llm_generated_summary"]
        }
    )

    # Update the state with the edited text
    return {
        "llm_generated_summary": result["edited_text"] 
    }

# Add the node to the graph in an appropriate location
# and connect it to the relevant nodes.
graph_builder.add_node("human_editing", human_editing)
graph = graph_builder.compile(checkpointer=checkpointer)

...

# After running the graph and hitting the interrupt, the graph will pause.
# Resume it with the edited text.
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(
    Command(resume={"edited_text": "The edited text"}), 
    config=thread_config
)
```

자세한 예제는 [How to wait for user input using interrupt (interrupt를 사용하여 사용자 입력을 기다리는 방법)](../how-tos/human_in_the_loop/wait-user-input.ipynb)를 참조하십시오.

### Review Tool Calls (도구 호출 검토)

<figure markdown="1">
![image](img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
<figcaption>A human can review and edit the output from the LLM before proceeding. This is particularly
critical in applications where the tool calls requested by the LLM may be sensitive or require human oversight.
</figcaption>
</figure>

```python
def human_review_node(state) -> Command[Literal["call_llm", "run_tool"]]:
    # This is the value we'll be providing via Command(resume=<human_review>)
    human_review = interrupt(
        {
            "question": "Is this correct?",
            # Surface tool calls for review
            "tool_call": tool_call
        }
    )

    review_action, review_data = human_review

    # Approve the tool call and continue
    if review_action == "continue":
        return Command(goto="run_tool")

    # Modify the tool call manually and then continue
    elif review_action == "update":
        ...
        updated_msg = get_updated_msg(review_data)
        # Remember that to modify an existing message you will need
        # to pass the message with a matching ID.
        return Command(goto="run_tool", update={"messages": [updated_message]})

    # Give natural language feedback, and then pass that back to the agent
    elif review_action == "feedback":
        ...
        feedback_msg = get_feedback_msg(review_data)
        return Command(goto="call_llm", update={"messages": [feedback_msg]})
```

자세한 예제는 [how to review tool calls (도구 호출 검토 방법)](../how-tos/human_in_the_loop/review-tool-calls.ipynb)를 참조하십시오.

### Multi-turn conversation (다중 턴 대화)

<figure markdown="1">
![image](img/human_in_the_loop/multi-turn-conversation.png){: style="max-height:400px"}
<figcaption>A <strong>multi-turn conversation</strong> architecture where an <strong>agent</strong> and <strong>human node</strong> cycle back and forth until the agent decides to hand off the conversation to another agent or another part of the system.
</figcaption>
</figure>

**Multi-turn conversation (다중 턴 대화)**은 에이전트와 사람 간의 여러 번의 상호 작용을 포함하며, 이를 통해 에이전트는 대화 방식으로 사람으로부터 추가 정보를 수집할 수 있습니다.

이 디자인 패턴은 [multiple agents (다중 에이전트)](./multi_agent.md)로 구성된 LLM 애플리케이션에서 유용합니다. 하나 이상의 에이전트가 사람과 다중 턴 대화를 수행해야 할 수 있으며, 여기서 사람은 대화의 여러 단계에서 입력 또는 피드백을 제공합니다. 단순성을 위해 아래 에이전트 구현은 단일 노드로 설명되지만 실제로는 여러 노드로 구성된 더 큰 그래프의 일부일 수 있으며 조건부 에지를 포함할 수 있습니다.

=== "Using a human node per agent (에이전트당 휴먼 노드 사용)"

    이 패턴에서는 각 에이전트가 사용자 입력을 수집하기 위한 자체 휴먼 노드를 갖습니다.
    이는 휴먼 노드에 고유한 이름("에이전트 1용 휴먼", "에이전트 2용 휴먼" 등)을 지정하거나
    서브 그래프가 휴먼 노드와 에이전트 노드를 포함하는 서브 그래프를 사용하여 달성할 수 있습니다.

    ```python
    from langgraph.types import interrupt

    def human_input(state: State):
        human_message = interrupt("human_input")
        return {
            "messages": [
                {
                    "role": "human",
                    "content": human_message
                }
            ]
        }

    def agent(state: State):
        # Agent logic
        ...

    graph_builder.add_node("human_input", human_input)
    graph_builder.add_edge("human_input", "agent")
    graph = graph_builder.compile(checkpointer=checkpointer)

    # After running the graph and hitting the interrupt, the graph will pause.
    # Resume it with the human's input.
    graph.invoke(
        Command(resume="hello!"),
        config=thread_config
    )
    ```


=== "Sharing human node across multiple agents (여러 에이전트에서 휴먼 노드 공유)"

    이 패턴에서는 단일 휴먼 노드가 여러 에이전트에 대한 사용자 입력을 수집하는 데 사용됩니다. 활성 에이전트는 상태에서 결정되므로 사용자 입력이 수집된 후 그래프는 올바른 에이전트로 라우팅할 수 있습니다.

    ```python
    from langgraph.types import interrupt

    def human_node(state: MessagesState) -> Command[Literal["agent_1", "agent_2", ...]]:
        """A node for collecting user input."""
        user_input = interrupt(value="Ready for user input.")

        # Determine the **active agent** from the state, so 
        # we can route to the correct agent after collecting input.
        # For example, add a field to the state or use the last active agent.
        # or fill in `name` attribute of AI messages generated by the agents.
        active_agent = ... 

        return Command(
            update={
                "messages": [{
                    "role": "human",
                    "content": user_input,
                }]
            },
            goto=active_agent,
        )
    ```

자세한 예제는 [how to implement multi-turn conversations (다중 턴 대화 구현 방법)](../how-tos/multi-agent-multi-turn-convo.ipynb)를 참조하십시오.

### Validating human input (사람의 입력 유효성 검사)

그래프 자체 내에서 (클라이언트 측이 아닌) 사람이 제공한 입력의 유효성을 검사해야 하는 경우 단일 노드 내에서 여러 interrupt 호출을 사용하여 이를 달성할 수 있습니다.

```python
from langgraph.types import interrupt

def human_node(state: State):
    """Human node with validation."""
    question = "What is your age?"

    while True:
        answer = interrupt(question)

        # Validate answer, if the answer isn't valid ask for input again.
        if not isinstance(answer, int) or answer < 0:
            question = f"'{answer} is not a valid age. What is your age?"
            answer = None
            continue
        else:
            # If the answer is valid, we can proceed.
            break
            
    print(f"The human in the loop is {answer} years old.")
    return {
        "age": answer
    }
```

## The `Command` primitive (`Command` 기본 요소)

`interrupt` 함수를 사용하면 그래프가 interrupt에서 일시 중지되고 사용자 입력을 기다립니다.

Graph execution (그래프 실행)은 `invoke`, `ainvoke`, `stream` 또는 `astream` 메서드를 통해 전달할 수 있는 [Command](../reference/types.md#langgraph.types.Command) 기본 요소를 사용하여 재개할 수 있습니다.

`Command` 기본 요소는 재개 중에 그래프의 상태를 제어하고 수정할 수 있는 여러 옵션을 제공합니다.

1. **Pass a value to the `interrupt` (`interrupt`에 값 전달)**: `Command(resume=value)`를 사용하여 사용자 응답과 같은 데이터를 그래프에 제공합니다. 실행은 `interrupt`가 사용된 노드의 시작 부분부터 재개되지만 이번에는 `interrupt(...)` 호출이 그래프를 일시 중지하는 대신 `Command(resume=value)`에 전달된 값을 반환합니다.

       ```python
       # Resume graph execution with the user's input.
       graph.invoke(Command(resume={"age": "25"}), thread_config)
       ```

2. **Update the graph state (그래프 상태 업데이트)**: `Command(update=update)`를 사용하여 그래프 상태를 수정합니다. 재개가 `interrupt`가 사용된 노드의 시작 부분부터 시작된다는 점에 유의하십시오. 실행은 `interrupt`가 사용된 노드의 시작 부분부터 재개되지만 업데이트된 상태로 재개됩니다.

      ```python
      # Update the graph state and resume.
      # You must provide a `resume` value if using an `interrupt`.
      graph.invoke(Command(update={"foo": "bar"}, resume="Let's go!!!"), thread_config)
      ```

`Command`를 활용하여 그래프 실행을 재개하고, 사용자 입력을 처리하고, 그래프의 상태를 동적으로 조정할 수 있습니다.

## Using with `invoke` and `ainvoke` (`invoke` 및 `ainvoke`와 함께 사용)

`stream` 또는 `astream`을 사용하여 그래프를 실행하면 `interrupt`가 트리거되었음을 알리는 `Interrupt` 이벤트를 받게 됩니다.

`invoke` 및 `ainvoke`는 interrupt 정보를 반환하지 않습니다. 이 정보에 액세스하려면 `invoke` 또는 `ainvoke`를 호출한 후 [get_state](../reference/graphs.md#langgraph.graph.graph.CompiledGraph.get_state) 메서드를 사용하여 그래프 상태를 검색해야 합니다.

```python
# Run the graph up to the interrupt 
result = graph.invoke(inputs, thread_config)
# Get the graph state to get interrupt information.
state = graph.get_state(thread_config)
# Print the state values
print(state.values)
# Print the pending tasks
print(state.tasks)
# Resume the graph with the user's input.
graph.invoke(Command(resume={"age": "25"}), thread_config)
```

```pycon
{'foo': 'bar'} # State values
(
    PregelTask(
        id='5d8ffc92-8011-0c9b-8b59-9d3545b7e553', 
        name='node_foo', 
        path=('__pregel_pull', 'node_foo'), 
        error=None, 
        interrupts=(Interrupt(value='value_in_interrupt', resumable=True, ns=['node_foo:5d8ffc92-8011-0c9b-8b59-9d3545b7e553'], when='during'),), state=None, 
        result=None
    ),
) # Pending tasks. interrupts 
```

## How does resuming from an interrupt work? (interrupt에서 재개하는 방법)

!!! warning

    Resuming from an `interrupt` is **different** from Python's `input()` function, where execution resumes from the exact point where the `input()` function was called.

`interrupt` 사용의 중요한 측면은 재개 작동 방식을 이해하는 것입니다. `interrupt` 후 실행을 재개하면 그래프 실행이 마지막 `interrupt`가 트리거된 **graph node (그래프 노드)**의 **beginning (시작)**부터 시작됩니다.

노드의 시작부터 `interrupt`까지의 **All (모든)** 코드가 다시 실행됩니다.

```python
counter = 0
def node(state: State):
    # All the code from the beginning of the node to the interrupt will be re-executed
    # when the graph resumes.
    global counter
    counter += 1
    print(f"> Entered the node: {counter} # of times")
    # Pause the graph and wait for user input.
    answer = interrupt()
    print("The value of counter is:", counter)
    ...
```

**resuming (그래프 재개)** 시 카운터가 두 번째로 증가하여 다음 출력이 생성됩니다.

```pycon
> Entered the node: 2 # of times
The value of counter is: 2
```

## Common Pitfalls (일반적인 함정)

### Side-effects (부작용)

노드가 재개될 때마다 다시 트리거되므로 API 호출과 같은 side effect (부작용)가 있는 코드를 중복을 피하기 위해 `interrupt` **after (뒤)**에 배치하십시오.

=== "Side effects before interrupt (BAD) (interrupt 전의 부작용 (나쁨))"

    이 코드는 `interrupt`에서 노드가 재개될 때 API 호출을 다시 실행합니다.

    API 호출이 idempotent (멱등성)이 아니거나 비용이 많이 드는 경우 문제가 될 수 있습니다.

    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""
        api_call(...) # This code will be re-executed when the node is resumed.
        answer = interrupt(question)
    ```

=== "Side effects after interrupt (OK) (interrupt 후의 부작용 (좋음))"

    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""
        
        answer = interrupt(question)
        
        api_call(answer) # OK as it's after the interrupt
    ```

=== "Side effects in a separate node (OK) (별도 노드의 부작용 (좋음))"

    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""
        
        answer = interrupt(question)
        
        return {
            "answer": answer
        }

    def api_call_node(state: State):
        api_call(...) # OK as it's in a separate node
    ```

### Subgraphs called as functions (함수로 호출되는 서브 그래프)

[as a function (함수)](low_level.md#as-a-function)으로 서브 그래프를 호출할 때 **parent graph (상위 그래프)**는 서브 그래프가 호출된 (그리고 `interrupt`가 트리거된) **beginning of the node (노드 시작)**부터 실행을 재개합니다. 마찬가지로 **subgraph (서브 그래프)**는 `interrupt()` 함수가 호출된 **beginning of the node (노드 시작)**부터 재개됩니다.

예를 들어,

```python
def node_in_parent_graph(state: State):
    some_code()  # <-- This will re-execute when the subgraph is resumed.
    # Invoke a subgraph as a function.
    # The subgraph contains an `interrupt` call.
    subgraph_result = subgraph.invoke(some_input)
    ...
```

??? "**Example: Parent and Subgraph Execution Flow (예제: 상위 및 서브 그래프 실행 흐름)**"

      Say we have a parent graph with 3 nodes:

      **Parent Graph (상위 그래프)**: `node_1` → `node_2` (subgraph call (서브 그래프 호출)) → `node_3`

      And the subgraph has 3 nodes, where the second node contains an `interrupt`:

      **Subgraph (서브 그래프)**: `sub_node_1` → `sub_node_2` (`interrupt`) → `sub_node_3`

      When resuming the graph, the execution will proceed as follows:

      1. **Skip `node_1`** in the parent graph (already executed, graph state was saved in snapshot).
      2. **Re-execute `node_2`** in the parent graph from the start.
      3. **Skip `sub_node_1`** in the subgraph (already executed, graph state was saved in snapshot).
      4. **Re-execute `sub_node_2`** in the subgraph from the beginning.
      5. Continue with `sub_node_3` and subsequent nodes.

      Here is abbreviated example code that you can use to understand how subgraphs work with interrupts.
      It counts the number of times each node is entered and prints the count.

      ```python
      import uuid
      from typing import TypedDict

      from langgraph.graph import StateGraph
      from langgraph.constants import START
      from langgraph.types import interrupt, Command
      from langgraph.checkpoint.memory import MemorySaver


      class State(TypedDict):
         """The graph state."""
         state_counter: int


      counter_node_in_subgraph = 0

      def node_in_subgraph(state: State):
         """A node in the sub-graph."""
         global counter_node_in_subgraph
         counter_node_in_subgraph += 1  # This code will **NOT** run again!
         print(f"Entered `node_in_subgraph` a total of {counter_node_in_subgraph} times")

      counter_human_node = 0

      def human_node(state: State):
         global counter_human_node
         counter_human_node += 1 # This code will run again!
         print(f"Entered human_node in sub-graph a total of {counter_human_node} times")
         answer = interrupt("what is your name?")
         print(f"Got an answer of {answer}")


      checkpointer = MemorySaver()

      subgraph_builder = StateGraph(State)
      subgraph_builder.add_node("some_node", node_in_subgraph)
      subgraph_builder.add_node("human_node", human_node)
      subgraph_builder.add_edge(START, "some_node")
      subgraph_builder.add_edge("some_node", "human_node")
      subgraph = subgraph_builder.compile(checkpointer=checkpointer)


      counter_parent_node = 0

      def parent_node(state: State):
         """This parent node will invoke the subgraph."""
         global counter_parent_node

         counter_parent_node += 1 # This code will run again on resuming!
         print(f"Entered `parent_node` a total of {counter_parent_node} times")
  
         # Please note that we're intentionally incrementing the state counter
         # in the graph state as well to demonstrate that the subgraph update
         # of the same key will not conflict with the parent graph (until
         subgraph_state = subgraph.invoke(state)
         return subgraph_state


      builder = StateGraph(State)
      builder.add_node("parent_node", parent_node)
      builder.add_edge(START, "parent_node")

      # A checkpointer must be enabled for interrupts to work!
      checkpointer = MemorySaver()
      graph = builder.compile(checkpointer=checkpointer)

      config = {
         "configurable": {
            "thread_id": uuid.uuid4(),
         }
      }

      for chunk in graph.stream({"state_counter": 1}, config):
         print(chunk)

      print('--- Resuming ---')

      for chunk in graph.stream(Command(resume="35"), config):
         print(chunk)
      ```

      This will print out

      ```pycon
      --- First invocation ---
      In parent node: {'foo': 'bar'}
      Entered `parent_node` a total of 1 times
      Entered `node_in_subgraph` a total of 1 times
      Entered human_node in sub-graph a total of 1 times
      {'__interrupt__': (Interrupt(value='what is your name?', resumable=True, ns=['parent_node:0b23d72f-aaba-0329-1a59-ca4f3c8bad3b', 'human_node:25df717c-cb80-57b0-7410-44e20aac8f3c'], when='during'),)}

      --- Resuming ---
      In parent node: {'foo': 'bar'}
      Entered `parent_node` a total of 2 times
      Entered human_node in sub-graph a total of 2 times
      Got an answer of 35
      {'parent_node': None} 
      ```



### Using multiple interrupts (여러 interrupt 사용)

**single (단일)** 노드 내에서 여러 interrupt를 사용하는 것은 [validating human input (사람의 입력 유효성 검사)](#validating-human-input)과 같은 패턴에 유용할 수 있습니다. 그러나 동일한 노드에서 여러 interrupt를 사용하면 주의해서 처리하지 않으면 예기치 않은 동작이 발생할 수 있습니다.

노드에 여러 interrupt 호출이 포함된 경우 LangGraph는 노드를 실행하는 작업에 특정된 resume 값 목록을 유지합니다. 실행이 재개될 때마다 노드의 시작 부분에서 시작됩니다. 발생하는 각 interrupt에 대해 LangGraph는 작업의 resume 목록에 일치하는 값이 있는지 확인합니다. Matching (일치)은 **strictly index-based (엄격하게 인덱스 기반)**이므로 노드 내에서 interrupt 호출 순서가 중요합니다.

문제를 피하려면 실행 간에 노드의 구조를 동적으로 변경하지 마십시오. 여기에는 interrupt 호출을 추가, 제거 또는 재정렬하는 것이 포함되며, 이러한 변경으로 인해 인덱스가 일치하지 않을 수 있습니다. 이러한 문제는 종종 `Command(resume=..., update=SOME_STATE_MUTATION)`을 통해 상태를 변경하거나 전역 변수에 의존하여 노드의 구조를 동적으로 수정하는 것과 같은 비정상적인 패턴에서 발생합니다.

??? "Example of incorrect code (잘못된 코드 예제)"

    ```python
    import uuid
    from typing import TypedDict, Optional

    from langgraph.graph import StateGraph
    from langgraph.constants import START 
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import MemorySaver


    class State(TypedDict):
        """The graph state."""

        age: Optional[str]
        name: Optional[str]


    def human_node(state: State):
        if not state.get('name'):
            name = interrupt("what is your name?")
        else:
            name = "N/A"

        if not state.get('age'):
            age = interrupt("what is your age?")
        else:
            age = "N/A"
            
        print(f"Name: {name}. Age: {age}")
        
        return {
            "age": age,
            "name": name,
        }


    builder = StateGraph(State)
    builder.add_node("human_node", human_node)
    builder.add_edge(START, "human_node")

    # A checkpointer must be enabled for interrupts to work!
    checkpointer = MemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable": {
            "thread_id": uuid.uuid4(),
        }
    }

    for chunk in graph.stream({"age": None, "name": None}, config):
        print(chunk)

    for chunk in graph.stream(Command(resume="John", update={"name": "foo"}), config):
        print(chunk)
    ```

    ```pycon
    {'__interrupt__': (Interrupt(value='what is your name?', resumable=True, ns=['human_node:3a007ef9-c30d-c357-1ec1-86a1a70d8fba'], when='during'),)}
    Name: N/A. Age: John
    {'human_node': {'age': 'John', 'name': 'N/A'}}
    ```

## Additional Resources 📚

- [**Conceptual Guide: Persistence**](persistence.md#replay): Read the persistence guide for more context on replaying.
- [**How to Guides: Human-in-the-loop**](../how-tos/index.md#human-in-the-loop): Learn how to implement human-in-the-loop workflows in LangGraph.
- [**How to implement multi-turn conversations**](../how-tos/multi-agent-multi-turn-convo.ipynb): Learn how to implement multi-turn conversations in LangGraph.
