**题目要求：**

1. 创建一个`TodoList`类，表示家庭任务清单，包含以下内容：
   - **类属性**：`total_tasks`（所有待办事项的总数，初始值为0）、`total_complete_tasks`（所有已完成任务总数）
   - **实例属性**：`owner`（拥有者）、`task_type`（类别，例如“生活”，“工作”），`tasks`（任务列表，每个任务是一个字典：{"task": "任务名", "completed": False}）、`created_time`（创建时间，提示datetime.datetime.now()可以获取当前时间）
2. 在`__init__`方法中：
   - 初始化实例属性
   - 任务列表为空
3. 定义实例方法：
   - `add_task(self,task_name)`：添加新任务（更新tasks列表，并更新total_tasks+=1）
   - `complete_task(self,task_name)`：将指定任务标记为已完成（并更新total_complete_tasks+=1）
   - `show_tasks(self,show_all=True)`：显示单个任务清单中所有任务（如果show_all=False，只显示未完成的任务）
   - `delete_task(self,task_name)`：删除指定任务（更新tasks列表，并更新total_tasks-=1等）
   - `get_progress(self)`：计算并返回单个任务清单中完成任务的百分比
4. 定义类方法：
   - `get_total_tasks(cls)`：获取家庭所有待办事项的总数
   - `get_total_progress(cls)`：获取家庭所有任务完成百分比
5. 定义静态方法：
   - `is_valid_task(task_name)`：验证任务名称是否有效（长度1-50字符）
6. 创建2个待办事项列表（不同拥有者），并进行以下操作：
   - 分别添加几个个任务
   - 完成部分任务
   - 显示所有任务和只显示未完成任务
   - 查看任务完成进度
   - 删除一个任务
   - 使用静态方法验证1个任务名称
7. **动态操作练习**：
   - 为某个TodoList对象动态添加一个`priority`属性（优先级）
   - 动态添加一个实例方法`sort_by_name()`，按任务名称排序
   - 动态增加类属性`category`
   - 尝试动态删除某个对象的`show_tasks`方法，观察调用时的错误