---
name: research-company
description: 深度调研指定公司（薪资/面经/文化/技术栈），调用搜索和 LLM 生成详细报告
argument-hint: <company-name> <target-role>
allowed-tools: Bash(career-agent:*), WebSearch, WebFetch
---

# 公司深度调研

对指定公司进行全方位调研，覆盖薪资水平、面试经验、公司文化、技术栈等维度。

## 使用方式

```
/research-company <公司名> <目标岗位>
```

示例：
```
/research-company 字节跳动 "后端开发"
/research-company 小红书 "数据分析"
```

## 执行步骤

1. 使用 WebSearch 搜索 "<公司名> <岗位> 招聘 薪资" 获取薪资区间
2. 搜索 "<公司名> 工作体验 面经 优缺点" 获取文化和口碑
3. 搜索 "<公司名> 面试经验 <岗位>" 获取面试流程
4. 在知乎 (zhihu.com)、脉脉 (maimai.cn)、小红书 (xiaohongshu.com) 上交叉验证
5. 汇总为结构化报告：薪资区间、（首先对评价进行#情感分析判断满意度1-100分）正面/负面评价，分别对正面/负面评价进行总结并打分
6. 面试准备建议。

## 输出格式

```
## {公司名} | {目标岗位}

### 薪资参考
- 区间：XX 万- XX 万/年
- 依据：公开招聘信息

### 公司文化
**优点：**
- ...
**风险：**
- ...

### 面试经验
- ...

### 准备建议
- ...
```
