---
name: analyze-resume
description: 分析简历文件，解析技能/经验/教育背景，并生成匹配公司报告
argument-hint: <resume-file> <target-role>
allowed-tools: Bash(career-agent:*), Read
---

# 简历分析与公司匹配

对简历文件进行深度解析，然后发现并调研匹配的目标公司，生成完整报告。

## 使用方式

```
/analyze-resume <简历文件路径> <目标岗位>
```

示例：
```
/analyze-resume resume.pdf "Python 后端开发工程师"
/analyze-resume ~/简历.txt "数据分析师"
```

## 执行步骤

1. 确认简历文件存在且格式支持 (txt/md/pdf/docx)
2. 调用 `career-agent --resume <文件> --target <岗位>` 运行分析
3. 等待完成，检查输出的 JSON 报告
4. 总结报告要点：匹配度排名、薪资范围、面试经验、改进建议

## 可用的 CLI 参数

- `--resume <path>` — 简历文件路径
- `--target <role>` — 目标岗位描述
- `--max-companies <n>` — 最多调研几家公司 (5-10)
- `--no-search` — 离线模式，使用默认候选公司池
- `--out <path>` — 保存 JSON 报告到指定路径

## 输出说明

生成的 AgentReport 包含：
- `resume` — 解析后的简历画像
- `companies` — 公司列表（按匹配度降序），每家包含 match_score、薪资估计、优缺点（接受到的评论数据）、面经、定制简历建议
- `interview_preps` — 每家公司的面试准备（问题+策略）
- `caveats` — 数据来源和局限性说明
- `match_evidence` — 每个评分点的证据映射
