# 第1章 实现 DialogueEngine 入口

前面已经分别实现 Planning、任务处理、知识检索、闲聊和澄清模块。`DialogueEngine` 负责组织这些组件，完成一条用户消息从进入系统到生成处理结果的完整过程。

本章先定义 `DialogueEngine` 的依赖，再实现它对外提供的 `process_message()`。后续章节沿着该方法中的调用，逐层实现具体处理逻辑。

## 1.1 定义 DialogueEngine

在 `atguigu.engine.dialogue_engine.py` 模块中编写导入，并定义 `DialogueEngine`：

```python
import time

from atguigu.chitchat.handler import ChitchatHandler
from atguigu.clarify.responder import ClarifyResponder
from atguigu.domain.messages import (
    BotMessage,
    MessageType,
    ProcessResult,
    UserMessage,
)
from atguigu.domain.state import (
    DialogueState,
    FocusedObject,
    Turn,
)
from atguigu.knowledge.handler import KnowledgeHandler
from atguigu.plan.models import ClarifyReason
from atguigu.plan.turn_planner import TurnPlanner
from atguigu.plan.validator import TurnPlanValidator
from atguigu.task.command.models import (
    Command,
    SetSlotsCommand,
)
from atguigu.task.flow.models import FlowCatalog
from atguigu.task.flow.steps import CollectSlotStep
from atguigu.task.handler import TaskHandler


class DialogueEngine:
    def __init__(
        self,
        turn_planner: TurnPlanner,
        task_handler: TaskHandler,
        knowledge_handler: KnowledgeHandler,
        chitchat_handler: ChitchatHandler,
        clarify_responder: ClarifyResponder,
        turn_plan_validator: TurnPlanValidator,
    ) -> None:
        self.turn_planner = turn_planner
        self.task_handler = task_handler
        self.knowledge_handler = knowledge_handler
        self.chitchat_handler = chitchat_handler
        self.clarify_responder = clarify_responder
        self.turn_plan_validator = turn_plan_validator
```

`DialogueEngine` 通过构造方法接收以下组件：

| 属性 | 作用 |
|------|------|
| `turn_planner` | 理解文本消息并生成 `TurnPlan`。 |
| `turn_plan_validator` | 校验 `TurnPlan` 能否在当前状态下执行。 |
| `task_handler` | 处理任务 Command，并推进活动任务。 |
| `knowledge_handler` | 根据知识 Intent 检索信息并生成回复。 |
| `chitchat_handler` | 生成闲聊回复。 |
| `clarify_responder` | 根据澄清原因生成澄清回复。 |

这些组件都由外部创建并传入。`DialogueEngine` 只负责组织调用，不负责了解组件内部如何生成计划、推进 Flow 或调用大模型。

## 1.2 实现 process_message()

`process_message()` 是 `DialogueEngine` 对外提供的入口。继续在类中添加：

```python
async def process_message(
    self,
    state: DialogueState,
    user_message: UserMessage,
) -> ProcessResult:
    self._prepare_session(state)
    turn = Turn.create(user_message)

    if user_message.type is MessageType.TEXT:
        messages = await self._handle_text_message(
            state=state,
            user_message=user_message,
        )
    else:
        state.shared.set_focused_object(
            FocusedObject(
                type=user_message.object.type,
                id=user_message.object.id,
                title=user_message.object.title,
                attributes=dict(
                    user_message.object.attributes
                ),
            )
        )
        messages = await self._handle_object_message(
            message=user_message,
            state=state,
        )

    turn.bot_messages.extend(messages)
    state.shared.append_turn(turn)

    return ProcessResult(
        sender_id=user_message.sender_id,
        message_id=user_message.message_id,
        messages=messages,
    )
```

该方法按照以下顺序处理一条用户消息：

1. `_prepare_session()` 确保当前存在可用的 Session。
2. 使用用户消息创建本轮 `Turn`。
3. 根据 `MessageType` 选择文本消息或对象消息处理逻辑。
4. 将本轮产生的客服消息写入 `Turn`。
5. 将完整的 `Turn` 加入当前 Session。
6. 返回包含本轮客服消息的 `ProcessResult`。

对于对象消息，方法会先将 `MessageObject` 转换为 `FocusedObject`，记录到共享状态，再进行后续处理。具体实现将在第 4 章展开。

当前用户消息不会提前加入 Session。只有消息处理完成后，包含用户消息和全部客服回复的 `Turn` 才会一次性提交。具体实现将在第 5 章展开。

# 第2章 准备 Session

`process_message()` 首先调用 `_prepare_session()`，确保本轮消息可以归入一个活动 Session。

本章先实现 `DialogueEngine` 中的会话准备入口，再沿着该方法的调用，依次实现活动 Session 的管理、Session 的创建与关闭，以及新会话开始前的状态重置。

## 2.1 实现会话准备入口

继续在 `DialogueEngine` 类中添加：

```python
def _prepare_session(
    self,
    state: DialogueState,
) -> None:
    session = state.shared.current_session

    if session is None:
        state.shared.start_session()
    elif time.time() - session.last_activity_at > 60 * 60:
        state.shared.close_current_session()
        state.reset_runtime_state_for_new_session()
        state.shared.start_session()
```

会话准备分为三种情况：

| 当前状态 | 处理方式 |
|----------|----------|
| 不存在活动 Session | 创建一个新 Session。 |
| 活动 Session 未超过一小时 | 继续使用当前 Session。 |
| 活动 Session 已超过一小时 | 关闭旧 Session，清理运行状态，再创建新 Session。 |

`reset_runtime_state_for_new_session()` 会清理任务状态和当前聚焦对象。这些信息只对当前会话有效，不应继续影响新 Session。

`_prepare_session()` 只负责确保当前 Session 可用，不在这里更新最后活动时间。本轮消息处理完成并写入 Session 时，`Session.append_turn()` 会统一更新 `last_activity_at`。

旧 Session 不会被删除，它仍然保存在 `state.shared.sessions` 中。`current_session` 直接返回列表末尾的 Session。

下面沿着 `_prepare_session()` 使用的方法，逐层补充 Session 生命周期的实现。

## 2.2 管理活动 Session

`_prepare_session()` 首先通过 `current_session` 取得当前活动 Session。继续在 `atguigu.domain.state` 模块的 `SharedState` 类中添加：

```python
@property
def current_session(self) -> Session | None:
    if not self.sessions:
        return None
    return self.sessions[-1]
```

`sessions` 按照创建顺序保存会话，因此列表为空时返回 `None`，否则直接返回最后一个 Session。Session 关闭后会立即创建并追加新 Session，因此不需要再通过 `closed_at` 判断它是否为当前 Session。

`_prepare_session()` 使用 `start_session()` 创建新 Session，使用 `close_current_session()` 关闭当前 Session。继续在 `SharedState` 类中添加：

```python
def start_session(self) -> Session:
    session = Session.create()
    self.sessions.append(session)
    return session

def close_current_session(self) -> None:
    session = self.current_session
    if session is not None:
        session.close()
```

`start_session()` 只负责创建并追加新 Session，`close_current_session()` 只负责关闭当前 Session。`_prepare_session()` 负责组织二者的调用顺序，使关闭旧会话、清理运行状态和创建新会话三个操作清晰可见。

## 2.3 创建和关闭 Session

`SharedState` 将具体的创建和关闭操作交给 `Session` 自身完成。

首先在 `atguigu.domain.state` 模块中补充导入：

```python
import time
import uuid
```

然后在 `Session` 类中添加：

```python
@classmethod
def create(cls) -> "Session":
    now = time.time()
    return cls(
        session_id=str(uuid.uuid4()),
        started_at=now,
        last_activity_at=now,
    )

def close(self) -> None:
    self.closed_at = time.time()
```

`Session.create()` 生成唯一的 `session_id`，并使用当前时间初始化开始时间和最后活动时间。`close()` 不会删除 Session，只负责记录关闭时间；当前 Session 始终由它在 `sessions` 中的位置确定。

## 2.4 重置会话运行状态

Session 超时后，`_prepare_session()` 会调用 `reset_runtime_state_for_new_session()`，清理只对当前会话有效的运行状态。

继续在 `atguigu.domain.state` 模块的 `DialogueState` 类中添加：

```python
def reset_runtime_state_for_new_session(self) -> None:
    self.tasks.reset()
    self.shared.clear_focus()
```

该方法将任务状态和聚焦对象的清理分别交给对应的状态对象。

在 `atguigu.domain.task` 模块的 `TaskState` 类中添加：

```python
def reset(self) -> None:
    self.active = None
    self.paused.clear()
```

在 `atguigu.domain.state` 模块的 `SharedState` 类中添加：

```python
def clear_focus(self) -> None:
    self.focused_object = None
```

任务状态和聚焦对象都服务于当前会话。新 Session 创建后，用户仍然保留历史 Sessions，但不会继续处于旧会话的任务中，也不会继续关注旧会话中的商品或订单。

# 第3章 处理文本消息

文本消息需要先经过 Planning，再根据有效计划进入任务、知识检索或闲聊处理。

## 3.1 实现文本消息入口

继续在 `DialogueEngine` 类中添加 `_handle_text_message()`：

```python
async def _handle_text_message(
    self,
    state: DialogueState,
    user_message: UserMessage,
) -> list[BotMessage]:
    turn_plan = await self.turn_planner.predict(
        state,
        user_message,
        self.task_handler.flows,
        self.knowledge_handler.knowledge_intents,
    )
    validation = self.turn_plan_validator.validate(
        turn_plan,
        state=state,
        flows=self.task_handler.flows,
        knowledge_intents=(
            self.knowledge_handler.knowledge_intents
        ),
    )

    if not validation.valid:
        return await self.clarify_responder.respond(
            state=state,
            user_message=user_message,
            reason=validation.reason,
        )

    if turn_plan.task is not None:
        return await self.task_handler.handle(
            commands=turn_plan.task.commands,
            state=state,
            user_message=user_message,
        )

    elif turn_plan.knowledge is not None:
        if state.tasks.active:
            state.tasks.suspend_active()
        return await self.knowledge_handler.handle(
            state=state,
            user_message=user_message,
            intents=turn_plan.knowledge.intents,
        )

    if state.tasks.active:
        state.tasks.suspend_active()
    return await self.chitchat_handler.handle(
        state=state,
        user_message=user_message,
    )
```

该方法分为计划生成、计划校验和计划执行三个阶段。

## 3.2 生成并校验计划

`TurnPlanner.predict()` 需要当前状态、用户消息和系统能力：

| 信息 | 取得位置 |
|------|----------|
| 当前状态 | `state` |
| 当前用户消息 | `user_message` |
| 系统支持的任务 | `self.task_handler.flows` |
| 系统支持的知识意图 | `self.knowledge_handler.knowledge_intents` |

Planner 根据这些信息生成 `TurnPlan`，Validator 再使用相同的系统能力检查计划是否可执行。

校验失败时，方法不再执行任何处理方向，而是将 `validation.reason`、当前状态和用户消息交给 `ClarifyResponder`，直接返回澄清回复。

## 3.3 执行有效计划

校验通过后，`TurnPlan` 只会包含一个处理方向：

| 计划方向 | 处理方式 |
|----------|----------|
| `task` | 将 `commands` 交给 `TaskHandler`。 |
| `knowledge` | 暂停活动任务，再将 `intents` 交给 `KnowledgeHandler`。 |
| `chitchat` | 暂停活动任务，再调用 `ChitchatHandler`。 |

任务方向可能包含启动、恢复、取消或补充任务信息等操作，因此任务状态由 `TaskHandler` 根据 Command 处理，`DialogueEngine` 不会提前暂停活动任务。

知识检索和闲聊表示用户暂时离开当前任务。如果存在活动任务，`suspend_active()` 会将它移入暂停任务列表，使用户之后仍然可以通过任务 Command 恢复。

前两个方向都不满足时，可以按照计划已经通过校验处理，此时剩余的唯一方向就是 `chitchat`，因此代码直接进入闲聊处理。

# 第4章 处理对象消息

对象消息表示用户从页面中选择了一个商品或订单。`DialogueEngine` 先将消息携带的对象记录为当前聚焦对象，再判断它能否直接补充活动任务。

## 4.1 记录聚焦对象

`UserMessage.object` 表示本轮消息携带的对象，`state.shared.focused_object` 表示后续对话可以继续使用的当前聚焦对象。`process_message()` 通过以下代码完成转换：

```python
state.shared.set_focused_object(
    FocusedObject(
        type=user_message.object.type,
        id=user_message.object.id,
        title=user_message.object.title,
        attributes=dict(
            user_message.object.attributes
        ),
    )
)
```

`attributes` 使用 `dict()` 创建新字典，避免用户消息和对话状态共同持有同一个可变字典。

继续在 `atguigu.domain.state` 模块的 `SharedState` 类中添加：

```python
def set_focused_object(
    self,
    focused_object: FocusedObject,
) -> None:
    self.focused_object = focused_object
```

聚焦对象记录到共享状态后，任务处理和知识检索都可以在后续消息中继续使用它。

## 4.2 实现对象消息入口

继续在 `DialogueEngine` 类中添加 `_handle_object_message()`：

```python
async def _handle_object_message(
    self,
    message: UserMessage,
    state: DialogueState,
) -> list[BotMessage]:
    command = self._object_slot_command(
        message=message,
        state=state,
        flows=self.task_handler.flows,
    )

    if command is None:
        return await self.clarify_responder.respond(
            state=state,
            user_message=message,
            reason=ClarifyReason.OBJECT_REQUIRES_INTENT,
        )

    return await self.task_handler.handle(
        commands=[command],
        state=state,
        user_message=message,
    )
```

对象消息有两种处理结果：

1. 对象可以补充活动任务正在收集的 Slot，将其转换为 `SetSlotsCommand`，再交给 `TaskHandler`。
2. 对象不能直接补充当前任务，使用 `OBJECT_REQUIRES_INTENT` 生成澄清回复，询问用户希望对该对象执行什么操作。

## 4.3 将对象转换为 Command

`_object_slot_command()` 负责判断对象类型与当前正在收集的 Slot 是否匹配。继续在 `DialogueEngine` 类中添加：

```python
def _object_slot_command(
    self,
    message: UserMessage,
    state: DialogueState,
    flows: FlowCatalog,
) -> Command | None:
    message_object = message.object
    object_type = message_object.type.strip().lower()
    collect_slot_name = self._current_collect_slot_name(
        state=state,
        flows=flows,
    )

    if (
        object_type == "order"
        and collect_slot_name == "order_number"
    ):
        return SetSlotsCommand(
            command="set_slots",
            slots={"order_number": message_object.id},
        )

    if (
        object_type == "product"
        and collect_slot_name == "product_id"
    ):
        return SetSlotsCommand(
            command="set_slots",
            slots={"product_id": message_object.id},
        )

    return None
```

当前支持以下转换规则：

| 对象类型 | 当前收集的 Slot | 生成的 Slot 数据 |
|----------|-----------------|------------------|
| `order` | `order_number` | `{"order_number": object.id}` |
| `product` | `product_id` | `{"product_id": object.id}` |

例如，活动任务正在收集 `order_number`，用户选择了以下订单：

```json
{
  "type": "order",
  "id": "O1001",
  "title": "订单 O1001"
}
```

方法会生成：

```python
SetSlotsCommand(
    command="set_slots",
    slots={"order_number": "O1001"},
)
```

`TaskHandler` 执行该 Command 后，会将订单号写入活动任务并继续推进 Flow。

## 4.4 取得当前收集的 Slot

对象转换还需要知道活动任务当前是否正在收集信息，以及正在收集哪个 Slot。继续在 `DialogueEngine` 类中添加：

```python
@staticmethod
def _current_collect_slot_name(
    state: DialogueState,
    flows: FlowCatalog,
) -> str | None:
    active_task = state.tasks.active
    if active_task is None:
        return None

    current_flow = flows.get_flow(active_task.flow_id)
    current_step = current_flow.get_step(
        active_task.step_id
    )
    if not isinstance(current_step, CollectSlotStep):
        return None

    return current_step.slot_name
```

该方法按照以下顺序查找：

1. 读取当前活动任务；没有活动任务时返回 `None`。
2. 根据活动任务的 `flow_id` 取得 Flow。
3. 根据活动任务的 `step_id` 取得当前 Step。
4. 当前 Step 不是 `CollectSlotStep` 时返回 `None`。
5. 返回 Collect Step 正在收集的 `slot_name`。

只有对象类型与 `slot_name` 同时匹配时，对象消息才会直接转换为 Command。其他情况只记录聚焦对象并生成澄清回复。

# 第5章 记录对话轮次

`process_message()` 使用 `Turn` 记录本轮用户消息和本轮产生的全部客服消息。Turn 在消息处理前创建，在消息处理完成后一次性加入当前 Session。

## 5.1 创建 Turn

`process_message()` 通过以下代码创建本轮 Turn：

```python
turn = Turn.create(user_message)
```

在 `atguigu.domain.state` 模块的 `Turn` 类中添加：

```python
@classmethod
def create(cls, user_message: UserMessage) -> "Turn":
    return cls(
        turn_id=str(uuid.uuid4()),
        user_message=user_message,
        bot_messages=[],
    )
```

`Turn.create()` 为本轮对话生成唯一的 `turn_id`，保存当前用户消息，并使用空列表初始化客服消息。此时本轮消息还没有加入 Session，各个处理组件读取到的历史记录不包含正在处理的当前轮次。

## 5.2 提交 Turn

文本消息或对象消息处理完成后，`process_message()` 将产生的客服消息写入 Turn，再将完整的 Turn 提交到当前 Session：

```python
turn.bot_messages.extend(messages)
state.shared.append_turn(turn)
```

`extend()` 将本轮产生的全部 `BotMessage` 加入 `turn.bot_messages`。继续在 `atguigu.domain.state` 模块的 `SharedState` 类中添加：

```python
def append_turn(self, turn: Turn) -> None:
    session = self.current_session
    if session is None:
        raise RuntimeError(
            "Cannot append a turn without an active session."
        )
    session.append_turn(turn)
```

`SharedState.append_turn()` 取得当前活动 Session，再将具体的追加操作交给该 Session。`process_message()` 已经在开始时准备了活动 Session，因此提交 Turn 时可以按照 Session 一定存在处理。

继续在 `Session` 类中添加：

```python
def append_turn(self, turn: Turn) -> None:
    self.turns.append(turn)
    self.last_activity_at = time.time()
```

`Session.append_turn()` 保存完整 Turn，并更新 Session 的最后活动时间。至此，本轮用户消息和由它产生的全部客服回复作为一个整体进入会话历史。

