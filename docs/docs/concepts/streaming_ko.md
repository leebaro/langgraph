# 스트리밍 (Streaming)

최종 사용자를 위한 반응형 앱을 구축하고 계십니까? 실시간 업데이트는 앱 진행 상황에 따라 사용자의 참여를 유지하는 데 중요합니다.

스트리밍하려는 데이터에는 세 가지 주요 유형이 있습니다.

1. 워크플로 진행 상황 (Workflow progress) (예: 각 그래프 노드가 실행된 후 상태 업데이트 가져오기).
2. LLM 토큰 (tokens) 생성.
3. 사용자 정의 업데이트 (Custom updates) (예: "10/100 레코드 가져옴 (Fetched 10/100 records)").

## 스트리밍 그래프 출력 (`.stream` 및 `.astream`)

`.stream` 및 `.astream`은 그래프 실행에서 출력을 스트리밍하기 위한 동기 및 비동기 메서드입니다. 이러한 메서드를 호출할 때 여러 가지 모드를 지정할 수 있습니다 (예: `graph.stream(..., mode="...")`):

- [`"values"`](../how-tos/streaming.ipynb#values): 그래프의 각 단계 후 상태의 전체 값을 스트리밍합니다.
- [`"updates"`](../how-tos/streaming.ipynb#updates): 그래프의 각 단계 후 상태에 대한 업데이트를 스트리밍합니다. 동일한 단계에서 여러 업데이트가 수행되는 경우 (예: 여러 노드가 실행되는 경우) 이러한 업데이트는 개별적으로 스트리밍됩니다.
- [`"custom"`](../how-tos/streaming.ipynb#custom): 그래프 노드 내부에서 사용자 정의 데이터 (custom data)를 스트리밍합니다.
- [`"messages"`](../how-tos/streaming-tokens.ipynb): LLM 토큰 (tokens) 및 LLM이 호출되는 그래프 노드에 대한 메타데이터 (metadata)를 스트리밍합니다.
- [`"debug"`](../how-tos/streaming.ipynb#debug): 그래프 실행 전반에 걸쳐 가능한 많은 정보를 스트리밍합니다.

목록으로 전달하여 여러 스트리밍 모드 (streaming modes)를 동시에 지정할 수도 있습니다. 이렇게 하면 스트리밍된 출력은 튜플 `(stream_mode, data)`이 됩니다. 예:

```python
graph.stream(..., stream_mode=["updates", "messages"])
```

```
...
('messages', (AIMessageChunk(content='Hi'), {'langgraph_step': 3, 'langgraph_node': 'agent', ...}))
...
('updates', {'agent': {'messages': [AIMessage(content="Hi, how can I help you?")]}})
```

아래 시각화는 `values` 및 `updates` 모드 간의 차이점을 보여줍니다.

![values vs updates](../static/values_vs_updates.png)

## LangGraph 플랫폼 (Platform)

스트리밍은 LLM 애플리케이션이 최종 사용자에게 반응성이 좋게 느껴지도록 하는 데 매우 중요합니다. 스트리밍 실행을 생성할 때 스트리밍 모드는 API 클라이언트에 다시 스트리밍되는 데이터를 결정합니다. LangGraph 플랫폼은 5가지 스트리밍 모드를 지원합니다.

- `values`: 각 [슈퍼 스텝 (super-step)](https://langchain-ai.github.io/langgraph/concepts/low_level/#graphs)이 실행된 후 그래프의 전체 상태를 스트리밍합니다. 값 스트리밍에 대한 [방법 안내서](../cloud/how-tos/stream_values.md)를 참조하십시오.
- `messages-tuple`: 노드 내부에서 생성된 모든 메시지에 대해 LLM 토큰을 스트리밍합니다. 이 모드는 주로 채팅 애플리케이션에 전원을 공급하기 위한 것입니다. 메시지 스트리밍에 대한 [방법 안내서](../cloud/how-tos/stream_messages.md)를 참조하십시오.
- `updates`: 각 노드가 실행된 후 그래프의 상태에 대한 업데이트를 스트리밍합니다. 업데이트 스트리밍에 대한 [방법 안내서](../cloud/how-tos/stream_updates.md)를 참조하십시오.
- `debug`: 그래프 실행 전반에 걸쳐 디버그 이벤트 (debug events)를 스트리밍합니다. 디버그 이벤트 스트리밍에 대한 [방법 안내서](../cloud/how-tos/stream_debug.md)를 참조하십시오.
- `events`: 그래프 실행 중에 발생하는 모든 이벤트 (그래프의 상태 포함)를 스트리밍합니다. 이벤트 스트리밍에 대한 [방법 안내서](../cloud/how-tos/stream_events.md)를 참조하십시오. 이 모드는 대규모 LCEL 애플리케이션을 LangGraph로 마이그레이션하는 사용자에게만 유용합니다. 일반적으로 이 모드는 대부분의 애플리케이션에 필요하지 않습니다.

여러 스트리밍 모드를 동시에 지정할 수도 있습니다. 여러 스트리밍 모드를 동시에 구성하는 방법에 대한 [방법 안내서](../cloud/how-tos/stream_multiple.md)를 참조하십시오.

스트리밍 실행을 생성하는 방법에 대한 [API 참조](../cloud/reference/api/api_ref.html#tag/threads-runs/POST/threads/{thread_id}/runs/stream)를 참조하십시오.

[이전 섹션](#스트리밍-그래프-출력-stream-및-astream)

[이전 섹션](#스트리밍-그래프-출력-stream-및-astream)

전송된 모든 이벤트에는 두 가지 속성이 있습니다.

- `event`: 이벤트 이름입니다.
- `data`: 이벤트와 관련된 데이터입니다.