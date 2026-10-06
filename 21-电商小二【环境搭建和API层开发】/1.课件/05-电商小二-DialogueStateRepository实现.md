# 第1章 DialogueStateRepository 的作用

`DialogueState` 只存在于一次请求的内存中。为了在下一次请求到来时继续使用之前的对话上下文，需要将它保存到数据库。

`DialogueStateRepository` 负责对话状态的读取和保存，对外提供以下两个方法：

| 方法 | 作用 |
|------|------|
| `load_state()` | 根据用户 ID 加载 `DialogueState`。 |
| `save_state()` | 保存最新的 `DialogueState`。 |

每个用户在数据库中对应一条状态记录：

```text
sender_id -> DialogueState
```

其中，`sender_id` 作为记录的主键，完整的 `DialogueState` 以 JSON 字符串保存。



# 第2章 ORM 类型与类型转换

## 2.1 ORM 类型定义

### 2.1.2 Base

所有的ORM类型都需要继承 SQLAlchemy 的声明式基类 `Base`。

在 `atguigu.models` 包下创建 `base.py` 文件，并编写如下代码：

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

### 2.1.1 DialogueStateRecord

`DialogueStateRecord` 表示数据库中的一条对话状态记录。

在 `atguigu.models` 包下创建 `dialogue_state.py` 文件，并编写如下代码：

```python
from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column

from atguigu.models.base import Base


class DialogueStateRecord(Base):
    __tablename__ = "dialogue_states"

    sender_id: Mapped[str] = mapped_column(
        String(255),
        primary_key=True,
    )
    state_json: Mapped[str] = mapped_column(
        Text,
        nullable=False,
    )
```

它包含以下两个字段：

| 字段 | 数据库类型 | 说明 |
|------|------------|------|
| `sender_id` | `VARCHAR(255)` | 用户唯一标识，同时作为主键。 |
| `state_json` | `TEXT` | 完整的对话状态 JSON。 |

## 2.2 类型转换

Repository 使用 Pydantic 的 `TypeAdapter` 完成 `DialogueState` 与 JSON 之间的双向转换，具体用法可参考 [Pydantic TypeAdapter 文档](https://docs.pydantic.dev/latest/api/type_adapter/)。

主要使用以下两个方法：

| 方法 | 输入 | 输出 |
|------|------|------|
| `dump_json()` | `DialogueState` | JSON 字节串。 |
| `validate_json()` | JSON 字符串或字节串 | `DialogueState`。 |

具体用法如下：

### 2.2.1 创建 TypeAdapter

在 `atguigu.repository.dialogue_state_repository` 模块中为 `DialogueState` 创建 `TypeAdapter`：

```python
from pydantic import TypeAdapter

from atguigu.domain.state import DialogueState


DIALOGUE_STATE_ADAPTER = TypeAdapter(DialogueState)
```

### 2.2.2 DialogueState 转换为 JSON

`dump_json()` 将 `DialogueState` 转换成 JSON 字节串：

```python
state_bytes = DIALOGUE_STATE_ADAPTER.dump_json(state)
```

其返回值类型是 `bytes`。数据库中的 `state_json` 是字符串，因此保存前需要进行 UTF-8 解码：

```python
state_json = state_bytes.decode("utf-8")
```

也可以直接写成：

```python
state_json = DIALOGUE_STATE_ADAPTER.dump_json(
    state
).decode("utf-8")
```

转换结果包含 `DialogueState` 中的全局状态、任务状态和全部嵌套属性。

### 2.2.3 JSON 转换为 DialogueState

`validate_json()` 接收 JSON 字符串或字节串，并将其转换成 `DialogueState`：

```python
state = DIALOGUE_STATE_ADAPTER.validate_json(state_json)
```

转换完成后，JSON 中的 `shared`、`tasks`、`sessions`、`turns` 等内容会分别构造成对应的属性类型。

# 第3章 实现 load 和 save

ORM 类型和类型转换准备完成后，就可以实现 `DialogueStateRepository`。

## 3.1 定义 DialogueStateRepository

03 文档已经在 `atguigu.repository.dialogue_state_repository` 模块中定义了 Repository 的方法签名。现在添加所需的导入、类型适配器和构造方法：

```python
from pydantic import TypeAdapter
from sqlalchemy import select
from sqlalchemy.dialects.mysql import insert
from sqlalchemy.ext.asyncio import AsyncSession

from atguigu.domain.state import DialogueState
from atguigu.models.dialogue_state import DialogueStateRecord


DIALOGUE_STATE_ADAPTER = TypeAdapter(DialogueState)


class DialogueStateRepository:

    def __init__(self, session: AsyncSession) -> None:
        self.session = session
```

`session` 是 Repository 的唯一属性，用于执行异步数据库操作。它由外部创建并通过构造方法传入，Repository 不负责创建或关闭数据库 Session。

## 3.2 实现 load_state()

在 `DialogueStateRepository` 类中实现 `load_state()` 方法：

```python
async def load_state(
    self,
    sender_id: str,
) -> DialogueState:
    statement = select(DialogueStateRecord).where(
        DialogueStateRecord.sender_id == sender_id
    )
    result = await self.session.execute(statement)
    record = result.scalar_one_or_none()

    if record is None:
        return DialogueState(sender_id=sender_id)

    return DIALOGUE_STATE_ADAPTER.validate_json(
        record.state_json
    )
```

该方法按照以下顺序加载状态：

1. 根据 `sender_id` 查询 `DialogueStateRecord`。
2. 使用 `scalar_one_or_none()` 取得状态记录。
3. 如果记录不存在，创建新的 `DialogueState`。
4. 如果记录存在，使用 `TypeAdapter` 将 `state_json` 还原成 `DialogueState`。

新用户还没有数据库记录，此时：

```python
DialogueState(sender_id=sender_id)
```

会通过 04 文档中定义的默认值，自动获得空的 `SharedState` 和 `TaskState`。

## 3.3 实现 save_state()

继续在 `DialogueStateRepository` 类中实现 `save_state()` 方法：

```python
async def save_state(
    self,
    state: DialogueState,
) -> None:
    state_json = DIALOGUE_STATE_ADAPTER.dump_json(
        state
    ).decode("utf-8")

    statement = insert(DialogueStateRecord).values(
        sender_id=state.sender_id,
        state_json=state_json,
    )
    statement = statement.on_duplicate_key_update(
        state_json=statement.inserted.state_json
    )

    await self.session.execute(statement)
    await self.session.commit()
```

该方法按照以下顺序保存状态：

1. 使用 `TypeAdapter` 将 `DialogueState` 转换成 JSON 字符串。
2. 使用 `sender_id` 和 `state_json` 构造 INSERT 语句。
3. 如果 `sender_id` 已经存在，则更新原记录的 `state_json`。
4. 执行 SQL 并提交事务。

这里使用的是 MySQL Upsert：记录不存在时执行插入，记录已经存在时执行更新。因此，无论当前用户是否已经有状态记录，`DialogueService` 都只需要调用同一个 `save_state()` 方法。

至此，`DialogueStateRepository` 已经能够完成新状态创建、已有状态加载和最新状态保存。序列化职责由 Repository 层的 `TypeAdapter` 承担，`DialogueState` 及其子对象仍然只包含状态属性。
