---
sidebar_position: 4
slug: /04-the-embedding-layer
---

# The Embedding Layer

The embedding layer turns chunks and questions into vectors that the doc engine can compare quickly. It sits between model selection and retrieval, so the rest of the app asks for meaning-preserving vectors instead of hard-coding a provider. That split keeps the ingestion path and search path aligned even when providers expose different limits, batching rules, or output formats.

## Series map

- [01 anatomy of a query](./01-anatomy-of-query.md)
- [05 graphrag](./05-graphrag.md)
- [06 the canvas orchestrator](./06-the-canvas-orchestrator.md)
- [07 the doc engine abstraction](./07-the-doc-engine-abstraction.md)
- [09 about this site](./09-about-this-site.md)

## What the embedding layer does

`rag/llm/embedding_model.py` defines the common shape. `Base` gives every provider the same surface, `_batched_encode` drives batch requests and keeps input order stable, and `EmbeddingError` turns provider failures into one consistent error path. Each provider also owns its own ceiling logic: some models truncate aggressively before request time, some depend on server-side truncation, and some retry around transient failures. The effect stays the same: callers hand over text, and the provider decides how to fit it into the model's limits.

The catalog in that file includes providers such as `OpenAIEmbed`, `QWenEmbed`, `OllamaEmbed`, and `PerplexityEmbed`. `rag/llm/__init__.py` then discovers the modules dynamically, finds every class that exports `_FACTORY_NAME`, and populates `EmbeddingModel` and `RerankModel` by factory name. The rest of the app therefore selects a provider through a name such as `OpenAI` or `Ollama`, not through a concrete class import.

## How a dataset chooses its model

The dataset row carries the embedding choice in `embd_id`. `KnowledgebaseService` reads that field back, and the ingestion worker resolves it through `tenant_llm_service.py` and `llm_service.py` when it prepares a task. The call chain runs from the `KnowledgebaseService` record to tenant-scoped model config, then to `TenantLLMService.model_instance()`, and finally to `LLMBundle`, which resolves the concrete embedding class at call time.

That design matters because the same tenant can own several model families, but a dataset still needs one clear default. The setup guide at https://ragflow.io/docs/dev/configure_knowledge_base covers the UI step that writes the dataset model choice; the runtime path later uses the stored value instead of asking for a fresh selection on every parse.

`api/db/joint_services/tenant_model_service.py` adds a newer per-instance model configuration layer. It matters, but it does not replace the dataset story here. The dataset record still anchors the ingestion run, while the joint service gives the app a finer way to resolve which concrete instance backs a named model.

## Why the choice stays pinned

The doc engine stores vectors in fields whose names encode the vector size, such as `q_512_vec`, `q_768_vec`, `q_1024_vec`, and `q_1536_vec`. The mapping files keep those vectors beside the chunk text in the same record, so the engine treats embedding shape as part of the stored document, not as an afterthought. That structure makes one dataset effectively depend on one embedding dimensionality and, by extension, one embedding family at a time.

`rag/svr/task_executor.py` makes that dependency explicit. `do_handle_task` binds the embedding model, probes it to learn `vector_size`, passes that size into `init_kb`, and then uses the same size for every chunk it stores in that ingestion run. When the embedding model changes, the stored chunks no longer match the field the engine expects, so the dataset needs a fresh parse and fresh vectors. Re-embedding does not just improve quality; it rebuilds the storage shape that retrieval depends on.

## Where embedding happens

`rag/svr/task_executor.py:embedding` builds the text that reaches the model. It starts from `docnm_kwd` as the title hint, prefers `question_kwd` when the parser produced questions, falls back to `content_with_weight`, strips table tags from the text, and truncates before it calls the model in batches. The task executor manages the batch loop, while the provider classes apply their own request sizes, token ceilings, and retry behavior. Some providers let the server clip oversized input, while others trim or retry on the client side.

The query-time path mirrors that shape. `rag/nlp/search.py` sends the question through `Dealer.get_vector()`, which calls `encode_queries()`, converts the result into a `q_<dim>_vec` lookup, and feeds that into the retrieval query. The same dimension that the model returned at ingestion now decides which vector field retrieval reads at search time.

RAPTOR uses the same stored vectors and skips chunks that lack the expected vector field, which keeps older parses from breaking the graph build. GraphRAG follows a separate side path in `rag/graphrag/general/entity_embedding.py`, where node embeddings support graph-specific work rather than the document index itself. For that branch of the system, see [05 graphrag](./05-graphrag.md).

## Storage and reranking

The vector lives beside the chunk text in the doc engine record, not in a separate vector store. `conf/mapping.json` and `conf/infinity_mapping.json` show that pairing clearly: the text fields and the `q_<dim>_vec` fields travel together, so search can load content and vectors from the same chunk. That shared record layout keeps retrieval simple, but it also locks the data to the model family that created it.

`rag/llm/rerank_model.py` uses the same registry pattern as the embedding layer, yet it solves a different problem. A rerank model scores query and chunk pairs, then normalizes the result to a common `0..1` scale so the retrieval blend can compare providers fairly. It does not persist vectors; it helps search sort candidates after the embedding layer has already done the heavy lifting. For the retrieval side of that handoff, see [01 anatomy of a query](./01-anatomy-of-query.md).

## Where to look in the code

- `rag/llm/embedding_model.py` — shared embedding base, batch helper, truncation limits, and provider error handling.
- `rag/llm/__init__.py` — dynamic provider registry for embedding and rerank models.
- `api/db/services/knowledgebase_service.py` — dataset records and stored `embd_id` values.
- `api/db/services/tenant_llm_service.py` and `api/db/services/llm_service.py` — runtime model resolution and `LLMBundle`.
- `rag/svr/task_executor.py` — vector-size discovery, ingestion embedding, RAPTOR hooks, and re-embedding.
- `rag/nlp/search.py` — query embedding, `q_<dim>_vec` lookup, and retrieval flow.
- `conf/mapping.json` and `conf/infinity_mapping.json` — vector fields stored alongside chunk text.
- `rag/llm/rerank_model.py` and `rag/graphrag/general/entity_embedding.py` — reranking and GraphRAG consumers.