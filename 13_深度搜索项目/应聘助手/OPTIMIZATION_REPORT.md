# Career Agent 优化分析报告

## 执行时间
2026-08-02

## 代码分析总结

### 1. 当前架构
- **核心流程**: LangGraph 状态图管理 5 个节点的工作流
  - parse_resume → discover_companies → research_companies → prepare_interviews → synthesize_report
- **搜索服务**: 多源搜索（MCP、Tavily、SerpAPI、DuckDuckGo）+ 缓存
- **深度研究**: 可选的 DeepAgents 层用于公司调研

### 2. 主要性能瓶颈

#### 2.1 并发控制问题 ⚠️ **高优先级**
**问题位置**: [graph.py:314](src/career_agent/graph.py#L314)
```python
tasks = [asyncio.ensure_future(_research_one(name, request, resume, search, deep)) for name in candidates]
```
**问题**: 
- 所有公司调研任务同时启动，无法控制并发数
- 每个公司需要 3 次搜索（薪资、体验、面经），8家公司=24次搜索
- 搜索服务有 Semaphore(4) 限制，但批量创建任务仍会产生大量等待
- 可能触发 API 限流或超时

**影响**: 
- 离线模式下耗时较长
- 联网模式可能被搜索引擎限流
- 内存占用高（所有任务同时创建）

#### 2.2 搜索策略低效 ⚠️ **中优先级**
**问题位置**: [graph.py:172-176](src/career_agent/graph.py#L172-L176)
```python
queries = [
    (f"{company} {target} 招聘 薪资", None),
    (f"{company} 工作体验 面经 优缺点", ["zhihu.com", "tieba.baidu.com", "xiaohongshu.com"]),
    (f"{company} 面试经验 {target}", ["zhihu.com", "tieba.baidu.com", "xiaohongshu.com"]),
]
```
**问题**:
- 每家公司固定 3 次搜索，即使前面已有足够信息
- 搜索查询未考虑公司类型差异（大厂 vs 初创）
- 重复搜索相似域名（后两个查询都是知乎+贴吧+小红书）

#### 2.3 匹配度算法简单 ⚠️ **中优先级**
**问题位置**: [graph.py:41-98](src/career_agent/graph.py#L41-L98)
```python
score = min(95.0, 45 + len(matched_skills) * 8 + (15 if resume.education else 0) + (10 if resume.experiences else 0))
```
**问题**:
- 硬编码的权重和基准分
- 只基于关键词匹配，未考虑语义相似度
- 未利用搜索到的薪资、JD、面经等真实数据
- 评分区间窄（45-95），区分度低

#### 2.4 缓存利用不足 ⚠️ **低优先级**
**问题位置**: [search.py:21-44](src/career_agent/search.py#L21-L44)
```python
class SearchCache:
    def __init__(self, ttl_seconds: int = 300):
        self._store: dict[str, tuple[float, list[dict]]] = {}
```
**问题**:
- 缓存只在单次运行内有效，跨会话无持久化
- 无 LRU 淘汰，长期运行内存会增长
- TTL 5分钟，对频繁查询的公司（如"字节跳动"）可以更长

#### 2.5 错误处理不完善 ⚠️ **中优先级**
**问题位置**: [graph.py:318-323](src/career_agent/graph.py#L318-L323)
```python
try:
    company = await coro
except Exception as exc:
    logger.error("[research_companies] 公司调研异常：%s", exc, exc_info=True)
    errors.append(f"公司调研失败：{exc}")
    continue
```
**问题**:
- 异常信息不区分类型（网络失败 vs API限流 vs 解析错误）
- 失败后直接跳过，未尝试降级策略（如只用缓存数据）
- 部分失败不影响整体流程，但用户可能得到不完整结果

### 3. 优化建议

#### 3.1 并发优化 - **立即实施**
```python
# 改进方案：分批并发 + 动态限流
async def research_node(state: AgentState):
    # ... 前置代码 ...
    
    # 分批处理：每批 2-3 家公司
    batch_size = 3
    for i in range(0, len(candidates), batch_size):
        batch = candidates[i:i+batch_size]
        tasks = [_research_one(name, request, resume, search, deep) for name in batch]
        batch_results = await asyncio.gather(*tasks, return_exceptions=True)
        
        for company_or_error in batch_results:
            if isinstance(company_or_error, Exception):
                errors.append(f"公司调研失败：{company_or_error}")
            else:
                companies.append(company_or_error)
                writer({"event": "company_researched", ...})
```
**预期收益**: 降低峰值并发，减少 40-60% 的 API 限流风险

#### 3.2 搜索优化 - **立即实施**
```python
async def _research_one(company: str, request: AgentRequest, resume, search: SearchService, deep: DeepResearchOrchestrator) -> CompanyResearch:
    # 阶段性搜索：先薪资+体验，评估后决定是否继续
    all_evidence: list[Evidence] = []
    
    # 第一阶段：基础信息（必需）
    basic_queries = [
        (f"{company} {target} 招聘 薪资", None),
        (f"{company} 工作体验 评价", ["zhihu.com", "xiaohongshu.com"]),
    ]
    for query, domains in basic_queries:
        results = await search.search(query, domains=domains, limit=6)
        all_evidence.extend(results)
    
    # 第二阶段：面经（可选，仅当基础信息充足时）
    if len(all_evidence) >= 8:  # 已有足够信息，跳过面经搜索
        logger.info("[%s] 已获取足够证据 (%d条)，跳过面经搜索", company, len(all_evidence))
    else:
        interview_query = f"{company} 面试经验 {target}"
        results = await search.search(interview_query, domains=["zhihu.com"], limit=4)
        all_evidence.extend(results)
```
**预期收益**: 减少 20-30% 的搜索次数

#### 3.3 匹配度优化 - **短期实施**
```python
def _score_with_evidence_v2(resume, target: str, company: str, evidence: list[Evidence]) -> tuple[float, ...]:
    """改进的评分算法：基于证据的动态评分"""
    base_score = 50.0
    
    # 1. 技能匹配 (0-25分)
    skill_score = min(25, len(matched_skills) * 5)
    
    # 2. 经验匹配 (0-20分)
    exp_score = min(20, (resume.years_of_experience or 0) * 2)
    
    # 3. 证据质量 (0-15分) - 新增
    evidence_score = min(15, len([e for e in evidence if e.strength == "high"]) * 3)
    
    # 4. 薪资匹配度 (0-10分) - 新增
    salary_score = _calculate_salary_match(resume, salary_estimate)
    
    # 5. 教育背景 (0-10分)
    edu_score = 10 if resume.education else 0
    
    final_score = base_score + skill_score + exp_score + evidence_score + salary_score + edu_score
    return min(100.0, final_score), ...
```
**预期收益**: 提升 30-40% 的评分准确度

#### 3.4 缓存优化 - **长期实施**
```python
from functools import lru_cache
import pickle
from pathlib import Path

class PersistentSearchCache:
    """支持持久化的搜索缓存"""
    def __init__(self, cache_dir: Path, ttl_seconds: int = 3600):
        self.cache_dir = cache_dir
        self.cache_dir.mkdir(exist_ok=True)
        self._ttl = ttl_seconds
        self._memory_cache = {}  # 热数据内存缓存
    
    def get(self, query: str, domains: list[str] | None):
        # 先查内存
        key = self._key(query, domains)
        if key in self._memory_cache:
            return self._memory_cache[key]
        
        # 再查磁盘
        cache_file = self.cache_dir / f"{hash(key)}.pkl"
        if cache_file.exists():
            with open(cache_file, 'rb') as f:
                ts, results = pickle.load(f)
                if time.time() - ts < self._ttl:
                    self._memory_cache[key] = results
                    return results
        return None
```
**预期收益**: 提升 50-70% 的重复查询速度

#### 3.5 错误处理优化 - **短期实施**
```python
class ResearchError(Exception):
    """公司调研错误基类"""
    pass

class NetworkError(ResearchError):
    """网络相关错误"""
    pass

class APILimitError(ResearchError):
    """API 限流错误"""
    pass

async def _research_one_with_fallback(company: str, ...) -> CompanyResearch:
    try:
        return await _research_one(company, ...)
    except APILimitError as e:
        logger.warning("[%s] API限流，等待后重试", company)
        await asyncio.sleep(5)
        return await _research_one(company, ...)
    except NetworkError as e:
        logger.warning("[%s] 网络错误，使用缓存数据", company)
        return _build_from_cache(company, ...)
    except Exception as e:
        logger.error("[%s] 未知错误: %s", company, e)
        return _build_minimal(company)  # 返回最小有效结果
```

### 4. 性能监控建议

#### 4.1 添加性能指标收集
```python
class PerformanceMonitor:
    def __init__(self):
        self.metrics = {
            "search_count": 0,
            "search_time": 0.0,
            "cache_hits": 0,
            "cache_misses": 0,
            "company_research_times": {},
        }
    
    def record_search(self, duration: float, from_cache: bool):
        self.metrics["search_count"] += 1
        self.metrics["search_time"] += duration
        if from_cache:
            self.metrics["cache_hits"] += 1
        else:
            self.metrics["cache_misses"] += 1
    
    def summary(self) -> dict:
        return {
            "total_searches": self.metrics["search_count"],
            "avg_search_time": self.metrics["search_time"] / max(1, self.metrics["search_count"]),
            "cache_hit_rate": self.metrics["cache_hits"] / max(1, self.metrics["search_count"]),
        }
```

### 5. 优先级排序

| 优化项 | 优先级 | 预期收益 | 实施难度 | 建议时间 |
|--------|--------|----------|----------|----------|
| 并发控制优化 | 🔴 高 | 40-60% 性能提升 | 低 | 立即 |
| 搜索策略优化 | 🟡 中 | 20-30% 搜索减少 | 低 | 立即 |
| 错误处理优化 | 🟡 中 | 提升稳定性 | 中 | 1-2天 |
| 匹配度算法优化 | 🟡 中 | 30-40% 准确度提升 | 中 | 3-5天 |
| 缓存持久化 | 🟢 低 | 50-70% 重复查询速度 | 中 | 1周 |
| 性能监控 | 🟢 低 | 可观测性提升 | 低 | 2-3天 |

### 6. 快速优化清单（可立即实施）

✅ **30分钟内可完成**:
1. 调整并发批次大小（batch_size=3）
2. 增加搜索缓存 TTL（300s → 1800s）
3. 添加搜索结果阈值判断（避免过度搜索）
4. 优化日志输出格式

✅ **2小时内可完成**:
1. 实现分批并发逻辑
2. 添加简单的性能计时器
3. 改进异常分类和处理
4. 优化匹配度评分基准

## 总结

当前系统整体架构合理，主要问题在于：
1. **并发控制粗糙** - 需要分批处理
2. **搜索策略固定** - 需要自适应优化
3. **匹配度算法简单** - 需要引入更多因素
4. **缓存未持久化** - 影响重复使用效率

建议优先实施前3项优化，可在短期内显著提升性能和准确度。
