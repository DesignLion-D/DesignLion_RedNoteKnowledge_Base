---
title: "AI 记忆架构 · 配套代码资料"
article: article/2026-05-17-agentic-memory/index.md
type: resource
created: 2026-05-17
---

## 说明

本资料是「AI 记不住你，是记忆架构的问题，不是 AI 不够聪明」的配套内容，包含文章中未展开的完整代码实现。适合想动手搭建 Agent 记忆层的开发者。

技术栈：Python + OpenAI Embeddings + ChromaDB（向量数据库）

---

## 安装依赖

```bash
pip install chromadb openai anthropic python-dotenv
```

---

## MemoryStore：向量记忆存储

负责写入记忆（含向量化）和语义检索，是整个系统的基础层。

```python
import chromadb
from openai import OpenAI
from datetime import datetime
import json, uuid

class MemoryStore:
    """Persistent vector memory for an AI agent."""

    def __init__(self, agent_id: str, persist_dir: str = "./memory_db"):
        self.agent_id = agent_id
        self.openai = OpenAI()

        # ChromaDB 持久化到磁盘，重启后仍保留
        self.client = chromadb.PersistentClient(path=persist_dir)
        self.collection = self.client.get_or_create_collection(
            name=f"agent_{agent_id}_memories",
            metadata={"hnsw:space": "cosine"}
        )

    def _embed(self, text: str) -> list[float]:
        response = self.openai.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        return response.data[0].embedding

    def remember(self, content: str, memory_type: str = "general", metadata: dict = None) -> str:
        """存入记忆，返回记忆 ID。"""
        memory_id = str(uuid.uuid4())
        embedding = self._embed(content)

        meta = {
            "type": memory_type,
            "timestamp": datetime.utcnow().isoformat(),
            "agent_id": self.agent_id,
            **(metadata or {})
        }

        self.collection.add(
            ids=[memory_id],
            embeddings=[embedding],
            documents=[content],
            metadatas=[meta]
        )
        return memory_id

    def recall(self, query: str, k: int = 5, memory_type: str = None, min_relevance: float = 0.6) -> list[dict]:
        """检索与查询语义最相关的 k 条记忆。"""
        query_embedding = self._embed(query)
        where = {"type": memory_type} if memory_type else None

        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=k,
            where=where,
            include=["documents", "metadatas", "distances"]
        )

        memories = []
        for doc, meta, dist in zip(
            results["documents"][0],
            results["metadatas"][0],
            results["distances"][0]
        ):
            relevance = 1 - dist  # cosine distance → similarity
            if relevance >= min_relevance:
                memories.append({
                    "content": doc,
                    "metadata": meta,
                    "relevance": round(relevance, 3)
                })

        return sorted(memories, key=lambda x: x["relevance"], reverse=True)

    def forget(self, memory_id: str):
        """删除特定记忆（用于合规或清理过期数据）。"""
        self.collection.delete(ids=[memory_id])
```

---

## EpisodicLogger：情节记忆记录器

在 MemoryStore 之上增加任务结果日志层，让 Agent 能从自身历史中学习。

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class Episode:
    task: str
    approach: str
    outcome: str           # "success" | "partial" | "failure"
    duration_ms: int
    token_cost: int
    quality_score: float   # 0.0 – 1.0
    notes: str = ""
    error: Optional[str] = None

class EpisodicLogger:
    def __init__(self, memory_store: MemoryStore):
        self.store = memory_store

    def log(self, episode: Episode):
        """将一次任务结果存为可检索的情节记忆。"""
        doc = (
            f"Task: {episode.task}\n"
            f"Approach: {episode.approach}\n"
            f"Outcome: {episode.outcome}\n"
            f"Notes: {episode.notes}"
        )
        self.store.remember(
            content=doc,
            memory_type="episode",
            metadata={
                "outcome": episode.outcome,
                "quality_score": episode.quality_score,
                "duration_ms": episode.duration_ms,
                "token_cost": episode.token_cost,
            }
        )

    def recall_similar(self, task: str, k: int = 3) -> list[dict]:
        """检索与当前任务相似的历史情节。"""
        return self.store.recall(
            query=task,
            k=k,
            memory_type="episode",
            min_relevance=0.65
        )
```

---

## MemoryAugmentedAgent：完整示例

把上面两个模块组合成一个带记忆的 Agent，展示完整的"检索→注入→调用→写入"循环。

```python
import anthropic
import time

class MemoryAugmentedAgent:
    def __init__(self, agent_id: str):
        self.client = anthropic.Anthropic()
        self.memory = MemoryStore(agent_id)
        self.episodes = EpisodicLogger(self.memory)

    def _build_memory_context(self, user_message: str) -> str:
        memories = self.memory.recall(user_message, k=4)
        episodes = self.episodes.recall_similar(user_message, k=2)

        context_parts = []

        if memories:
            context_parts.append("## Relevant memories\n" +
                "\n".join([
                    f"- [{m['metadata']['type']}] {m['content']} (relevance: {m['relevance']})"
                    for m in memories
                ])
            )

        if episodes:
            context_parts.append("## Past similar tasks\n" +
                "\n".join([f"- {e['content'][:200]}..." for e in episodes])
            )

        return "\n\n".join(context_parts) if context_parts else ""

    def run(self, user_message: str) -> str:
        start = time.time()

        # 1. 检索相关记忆
        memory_context = self._build_memory_context(user_message)

        # 2. 将记忆注入系统提示
        system = "You are a helpful agent with memory.\n"
        if memory_context:
            system += f"\n\n{memory_context}"

        # 3. 调用模型
        response = self.client.messages.create(
            model="claude-opus-4-6",
            max_tokens=1024,
            system=system,
            messages=[{"role": "user", "content": user_message}]
        )
        answer = response.content[0].text
        duration = int((time.time() - start) * 1000)

        # 4. 将本次交互写入记忆
        self.memory.remember(
            content=f"User asked: {user_message[:200]}",
            memory_type="interaction"
        )

        # 5. 记录情节
        self.episodes.log(Episode(
            task=user_message[:200],
            approach="single-turn with memory retrieval",
            outcome="success",
            duration_ms=duration,
            token_cost=response.usage.input_tokens + response.usage.output_tokens,
            quality_score=1.0,
        ))

        return answer
```

---

## 记忆管理策略

### 策略一：时间衰减评分

越旧的记忆相关性越低。综合相关度、重要性、新鲜度三个维度打分，用于排序和淘汰。

```python
import math
from datetime import datetime

def memory_score(
    relevance: float,
    importance: float,
    created_at: datetime,
    recency_weight: float = 0.3,
    decay_factor: float = 0.995
) -> float:
    """
    来源：Generative Agents 论文（Park et al., 2023）
    平衡相关性、重要性、新鲜度三个维度。
    """
    hours_old = (datetime.utcnow() - created_at).total_seconds() / 3600
    recency = math.pow(decay_factor, hours_old)

    return (
        relevance * 0.4 +
        importance * 0.3 +
        recency * recency_weight
    )
```

### 策略二：写入时重要性评分

存储时让模型自评这条信息的重要程度，只存高分项，从源头过滤噪音。

```python
import re

async def score_importance(client, content: str) -> float:
    prompt = f"""Rate the importance of saving this for future interactions.
0.0 = trivial (greeting)
0.5 = moderately useful
1.0 = critical (preferences, errors, decisions)

Information: {content}
Reply with ONLY the number."""

    try:
        response = await client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=10,
            messages=[{"role": "user", "content": prompt}]
        )
        text = response.content[0].text.strip()
        match = re.search(r"[-+]?\d*\.\d+|\d+", text)
        if match:
            return max(0.0, min(1.0, float(match.group())))
    except Exception:
        pass

    return 0.5
```

### 策略三：定期合并重复记忆

定时任务（如每晚）将高度相似的记忆合并为单条摘要，类似人类睡眠中的记忆巩固机制。

```python
async def consolidate_memories(store: MemoryStore, similarity_threshold: float = 0.92):
    """将近似重复的记忆合并为单条摘要。"""
    all_mems = store.collection.get(include=["documents", "embeddings", "ids"])
    if not all_mems["ids"]:
        return

    visited = set()
    consolidated_docs = []

    for mem_id, doc, emb in zip(
        all_mems["ids"], all_mems["documents"], all_mems["embeddings"]
    ):
        if mem_id in visited:
            continue

        results = store.collection.query(
            query_embeddings=[emb],
            n_results=10,
            include=["documents", "distances"]
        )

        group = [doc]
        visited.add(mem_id)

        for res_id, res_doc, dist in zip(
            results["ids"][0], results["documents"][0], results["distances"][0]
        ):
            sim = 1.0 - dist
            if res_id != mem_id and res_id not in visited and sim >= similarity_threshold:
                group.append(res_doc)
                visited.add(res_id)

        if len(group) > 1:
            summary = await summarize_group(group)
            consolidated_docs.append(summary)
        else:
            consolidated_docs.append(doc)

    store.collection.delete(where={})
    for doc in consolidated_docs:
        await store.remember(doc)
```

---

## 向量数据库选型

| 工具 | 适用场景 | 特点 |
|------|---------|------|
| **ChromaDB** | 本地开发、原型验证 | 零配置，嵌入式，免费 |
| **pgvector** | 已有 PostgreSQL 的生产环境 | 无额外基础设施，SQL 原生查询 |
| **Pinecone** | 大规模生产，百万级向量 | 托管服务，延迟低，按量计费 |
| **Qdrant** | 需要自托管的大规模场景 | 开源，性能强，过滤能力好 |

**余弦相似度原理**（用于语义检索的底层计算）：

```python
import numpy as np

def cosine_similarity(a: list, b: list) -> float:
    """
    1.0  = 语义完全相同
    0.0  = 无关
    -1.0 = 语义相反
    """
    a, b = np.array(a), np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```
