### 题1：文件读取、字符串拆分

**任务**：编写函数`count_words_in_file(filename)`，读取指定文本文件，返回文件中的总单词数（单词以空格等分隔）。
**示例**：假设`test.txt`内容如下，则函数返回`4`。

```
Hello world
Hello Python
```

### 题2：文件写入、字典操作

**任务**：编写函数`save_contacts(contacts, filename)`，将通讯录字典（键为姓名，值为电话）保存到文件中，每行格式为`姓名:电话`。再编写`load_contacts(filename)`，从文件中读取并重建字典。

### 题3：文件逐行读取、字符串处理

**任务**：有一个日志文件`log.txt`，每行格式为`时间 - 级别 - 消息`，例如`2023-01-01 10:00 - ERROR - Disk full`。编写函数`count_log_levels(filename)`，统计并返回一个字典，记录每个日志级别（INFO、WARNING、ERROR）出现的次数。
**示例**：`{"ERROR": 5, "WARNING": 3, "INFO": 10}`

log.txt文件内容

```
2023-01-01 10:00 - ERROR - Disk full
2023-01-01 10:05 - INFO - System started
2023-01-01 10:10 - WARNING - Low memory
2023-01-01 10:15 - INFO - User logged in
2023-01-01 10:20 - ERROR - Connection timeout
```

