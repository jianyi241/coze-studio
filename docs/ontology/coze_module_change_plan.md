# Coze Studio × 轻量 Ontology 改造范围（给 DevBot）

日期：2026-09-11  
仓库：https://github.com/coze-dev/coze-studio  
本地：`/Users/chenhao/project/coze-studio`  
配套 schema：同目录 `ontology.yaml`  
总策略：**宿主平台不重写**；在知识入库 + 检索链上增加「图召回」一路，与现有向量/ES/NL2SQL 并行，再进 RRF；用 A/B 证明优于纯向量。

---

## 0. 需求边界（MVP）

**做**
1. 文档切片入库后，异步/同步抽取实体三元组 → 校验 → 写入图存储（按 knowledge_id 隔离）
2. 检索链增加 `ontologyRetrieveNode`（实体链接 + k-hop），结果转成与现有一致的 `RetrieveSlice`/`schema.Document`
3. pack 阶段附带「路径证据」字符串，便于 prompt 可解释与评测
4. Feature flag：`ONTOLOGY_RAG_ENABLED`（默认关，避免影响现网路径）
5. 离线/脚本评测：同一题集对比 `vector-only` vs `vector+ontology`

**不做（MVP）**
- 完整 OWL DL / HermiT
- 新前端大改（配置可用 env；UI 开关可 Phase 2）
- 新工作流节点类型（Phase 2 可选）
- 替换 Milvus/ES

---

## 1. 现有链路（插入点）

检索（已确认）：
```
knowledgeSVC.Retrieve
  → queryRewriteNode
  → Parallel(vector, es, nl2sql, passContext)
  → reRankNode (RRF)
  → packResults
```

Agent 侧：知识库检索节点 → `PackRetrieveResultInfo` → 填入 `{{ knowledge }}`。

关键目录（请以本地 tree 再确认文件名）：
| 区域 | 路径（预期） | 角色 |
|------|----------------|------|
| 知识领域 | `backend/domain/knowledge/` | Retrieve 编排、入库 |
| 应用层 | `backend/application/` 下 knowledge 相关 | API/用例 |
| 检索存储 | `backend/infra/.../document/searchstore/` | Milvus/ES Manager |
| Agent 装配 | `backend/domain/agent/`（knowledge retriever lambda） | 对话时召回 |
| 工作流知识节点 | `backend/domain/workflow/`（Knowledge 相关） | 工作流召回（Phase 2） |
| Prompt | `backend/conf/prompt/` | 可加 ontology 证据模板 |

---

## 2. 模块改造清单（按优先级）

### P0 — 必做（证明优越性的最小闭环）

#### M1. 新包：`backend/domain/knowledge/ontology/`（或 `bizpkg/ontology`）
- `schema.go`：加载 `ontology.yaml`（可先内嵌一份 demo）
- `validate.go`：domain/range + 关系白名单
- `store.go`：接口 `OntologyStore`（AddTriples / GetEntity / KHop / DeleteByDocument）
- `extract.go`：调用已有 ChatModel，按 JSON schema 抽实体与三元组
- `link.go`：name/aliases 实体链接
- 单测：非法边拒绝、同义合并、2-hop 路径

**实现建议（Go 侧，避免强依赖 Python RDFLib）**
- MVP 图存储：SQLite（边表）或内存+持久化 JSON；接口后可换 Neo4j
- 或：独立 sidecar Python（RDFLib）+ HTTP；**更推荐纯 Go 边表**，运维更简单

#### M2. 入库挂钩（知识写入完成后）
插入点：文档解析/切片写入 searchstore **成功之后**（index pipeline 尾部）
- 对每个 chunk：extract → validate → upsert graph
- 删除/更新文档时：`DeleteByDocument(document_id)`
- 失败策略：记日志 + quarantine，**不阻断**原向量入库（flag 控制是否严格）

#### M3. 检索链并行节点 `ontologyRetrieveNode`
插入点：`knowledgeSVC.Retrieve` 的 `compose.NewParallel()` **再加一路**
- 输入：改写后的 query + knowledge IDs
- 步骤：从 query 提实体候选 → link → k-hop（默认 2）→ 收集相关 `Document`/`chunk_id` → 转为 RetrieveSlice
- 与 vector/ES 一起进现有 `reRankNode`（RRF）；若 RRF 不好接「图分」，可给 ontology 路固定高权重或单独 merge 后再 RRF

#### M4. Pack / Prompt 证据
- `packResults` 或 Agent `PackRetrieveResultInfo`：对 ontology 命中附加  
  `evidence_path: Component:knowledge --partOf--> ...`  
- 无路径且仅低分向量命中时：可选「低置信提示」（完整拒答可先做在评测脚本，产品拒答 Phase 2）

#### M5. Feature flag + 配置
- env：`ONTOLOGY_RAG_ENABLED`, `ONTOLOGY_KHOP=2`, `ONTOLOGY_SCHEMA_PATH`
- 默认关闭；DevBot 本地 Docker 打开做演示

### P1 — 强烈建议（评测与可演示）

#### M6. 评测夹具 `backend/domain/knowledge/ontology/eval/` 或仓库外脚本
- 40 题：20 单跳 / 15 多跳 / 5 对抗（见 ontology.yaml CQ）
- 指标：Accuracy、Faithfulness（无路径却断言=幻觉）、Multi-hop Acc、Explainability（有 path 比例）
- Baseline：关 ontology 仅向量；Treatment：开混合

#### M7. Demo 语料
- 10～20 段「Coze/软件组件」说明文档（可手写），覆盖 dependsOn / constrainedBy / precedes
- 预置 15～30 条黄金三元组（可先手工灌库，再开抽取）

### P2 — 可选（扩大产品面）

#### M8. 工作流「知识库检索」节点
- 复用同一 Retrieve；若要单独「Ontology 检索」节点，按 wiki《新增工作流节点》前后端都改——**MVP 跳过**

#### M9. 前端知识库设置
- 「启用本体增强检索」开关；MVP 用 env 即可

#### M10. 知识库级 schema 上传
- 每库一份 ontology.yaml；MVP 全局一份 demo schema

---

## 3. 建议实施顺序（DevBot）

1. Docker 跑通 + 确认 `domain/knowledge` 里 Retrieve/Index 真实文件路径  
2. 落地 `OntologyStore` + 单测  
3. 手工灌黄金三元组 → 只接 `ontologyRetrieveNode` → 能召回 chunk  
4. 再接入 extract 挂钩入库  
5. RRF 融合 + pack 证据  
6. 跑 M6 A/B，把数字写进 README 片段  

---

## 4. 验收标准

- [ ] Flag 关闭时，行为与上游一致（回归）  
- [ ] Flag 打开时，CQ2/CQ3 类多跳题准确率相对纯向量有可报告提升  
- [ ] 对抗题（无路径）幻觉率下降或可解释拒绝  
- [ ] 答案可附带至少 1 条图路径证据  
- [ ] 非法三元组无法入库  

---

## 5. 给 New Bot / DevBot 的接口

- Schema 变更：只改 `ontology.yaml`，通知双方  
- DevBot 回传：实际文件路径列表 + PR/分支名 + A/B 数字  
- New Bot 后续可补：题集 JSON、抽取 prompt、中文演示脚本  

