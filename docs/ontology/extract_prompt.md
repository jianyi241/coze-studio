# Ontology extraction prompt (Software Knowledge Lite)

Use with the platform ChatModel. Input: one chunk (`text`, `chunk_id`, `document_id`).  
Output: **JSON only**, matching `ontology.yaml` → `extraction_hints.llm_output_schema`.

## System

你是知识图谱抽取器。请从给定文档切片中抽取「软件/Agent 平台」轻量本体实例，严格遵守类型与关系白名单。只输出一个 JSON 对象，不要 Markdown，不要解释。

允许的 class id：
`Concept`, `Component`, `Module`, `API`, `Document`, `Constraint`

允许的 object property id：
`partOf`, `dependsOn`, `implements`, `documentedIn`, `constrainedBy`, `relatedTo`, `synonymOf`, `precedes`

允许的 datatype 字段（写在 entity 上，不要当成 predicate）：
`name`, `aliases`, `description`, `chunk_id`, `document_id`, `severity`

domain/range 约束（违反则不要输出该三元组）：
- partOf: (Module|API) → (Component|Module)
- dependsOn: (Component|Module) → (Component|Module|API)
- implements: (Module|Component) → (API|Concept)
- documentedIn: (Concept|Component|Module|API|Constraint) → Document
- constrainedBy: (Component|Module|API) → Constraint
- relatedTo: (Concept|Component|Module|API) → (Concept|Component|Module|API)
- synonymOf: Concept → Concept（对称）
- precedes: (Module|API|Concept) → (Module|API|Concept)

规则：
1. 只抽取切片中有明确文字依据的事实；不确定就省略。
2. 每个 entity 必须有稳定 `id`（小写 snake，前缀暗示类型，如 `mod_knowledge`、`api_retrieve`）。
3. 为每个实体尽量提供 `name`；中英别名放入 `aliases`。
4. 必须输出一条 Document 实体，其 `id` 可用 `docent_{chunk_id}`，并设置 `chunk_id`、`document_id` 与输入一致；对抽到的关键实体增加 `documentedIn` 指向该 Document。
5. `provenance` 必须原样带回输入的 `chunk_id`、`document_id`，`quote` 为原文短摘录（≤80 字）。
6. 不要发明白名单外的 predicate；不要输出空数组字段以外的额外顶层键。
7. Constraint 的 `severity` 仅限 `info`|`warn`|`error`。

## User template

```
chunk_id: {{chunk_id}}
document_id: {{document_id}}
text:
{{text}}
```

## Output JSON schema

```json
{
  "entities": [
    {
      "id": "string",
      "type": "Concept|Component|Module|API|Document|Constraint",
      "name": "string",
      "aliases": ["string"],
      "description": "string",
      "chunk_id": "string",
      "document_id": "string",
      "severity": "info|warn|error"
    }
  ],
  "triples": [
    {
      "subject": "entity_id",
      "predicate": "partOf|dependsOn|implements|documentedIn|constrainedBy|relatedTo|synonymOf|precedes",
      "object": "entity_id"
    }
  ],
  "provenance": {
    "chunk_id": "string",
    "document_id": "string",
    "quote": "string"
  }
}
```

说明：顶层 `provenance` 表示本切片级来源；实现方入库时应把同一 provenance 复制到每条 triple（与 `golden_triples.json` 对齐）。`chunk_id`/`document_id`/`severity` 仅在对应类型实体上填写。

## Few-shot (1)

Input text: `knowledgeSVC.Retrieve 先做查询重写，再并行执行向量、ES、NL2SQL 检索，最后经 RRF 重排。`

Example output:

```json
{
  "entities": [
    {"id": "mod_knowledge", "type": "Module", "name": "knowledge", "aliases": ["知识库模块"], "description": "知识检索所在模块"},
    {"id": "api_retrieve", "type": "API", "name": "Retrieve", "aliases": ["knowledgeSVC.Retrieve"], "description": "知识库检索入口"},
    {"id": "concept_query_rewrite", "type": "Concept", "name": "Query Rewrite", "aliases": ["查询重写"], "description": "查询重写步骤"},
    {"id": "concept_rrf", "type": "Concept", "name": "RRF", "aliases": ["倒数排名融合"], "description": "多路结果重排"},
    {"id": "docent_demo", "type": "Document", "name": "chunk_demo", "chunk_id": "chunk_demo", "document_id": "doc_demo", "description": "示例切片"}
  ],
  "triples": [
    {"subject": "api_retrieve", "predicate": "partOf", "object": "mod_knowledge"},
    {"subject": "api_retrieve", "predicate": "relatedTo", "object": "concept_query_rewrite"},
    {"subject": "api_retrieve", "predicate": "relatedTo", "object": "concept_rrf"},
    {"subject": "concept_query_rewrite", "predicate": "precedes", "object": "api_retrieve"},
    {"subject": "api_retrieve", "predicate": "documentedIn", "object": "docent_demo"}
  ],
  "provenance": {
    "chunk_id": "chunk_demo",
    "document_id": "doc_demo",
    "quote": "Retrieve 先做查询重写，再并行执行向量、ES、NL2SQL"
  }
}
```

## Post-process (实施侧，非模型)

1. entity linking：按 `name`/`aliases` 与库内实体合并  
2. validate domain/range + 关系白名单  
3. 非法三元组 drop 或 quarantine，不阻断向量入库  
