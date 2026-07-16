# The Chunking Template Zoo

## Overview

Documents do not arrive as flat text. A paper, a manual, a resume, and an email all carry structure that helps retrieval when the chunker respects it and hurts retrieval when the chunker ignores it. RAGFlow chooses a chunking template from `parser_id`, then dispatches that choice through the canonical `FACTORY` registry in `rag/svr/task_executor.py`.

The official dataset guide lists the available templates in a reference table, but the design question sits one level deeper: why these shapes exist and how they share the same indexing contract. This page treats the 14 templates as a small family of structural strategies instead of a flat configuration list. For the user-facing table, see [/docs/guides/dataset/configure_knowledge_base.md](/docs/guides/dataset/configure_knowledge_base.md).

## The shared chunk contract

Each template lives in `rag/app/` and exposes a `chunk()` function that turns one source document into chunk dictionaries. The fields vary by genre, but the contract stays stable: the chunk carries the text that search should rank, the tokenized forms that support indexing, and enough structure to point back to the source location.

The most visible anchors are `content_with_weight`, `content_ltks`, and `content_sm_ltks` for text and search; `page_num_int` and `position_int` when layout matters; `image` when media must travel with the chunk; and template-specific markers such as `doc_type_kwd`, `mom`, and `mom_with_weight` when a child chunk needs to remember its parent text. `rag/app/naive.py`, `rag/app/presentation.py`, and `rag/app/email.py` show the pattern from three different angles: plain-text merging, page-level slide chunks, and body and attachment handling.

## The template zoo by strategy

### General token-budget path

`rag/app/naive.py` carries the default path for mixed documents. It walks through text with delimiter-aware merging, stops when the token budget reaches its target, and then emits chunks. That makes it the safe baseline for content that does not announce a stronger structure.

The same module also picks the PDF backend. `PARSERS` and `normalize_layout_recognizer` route PDF work across DeepDoc, plain text, and third-party backends such as MinerU, Docling, OpenDataLoader, TCADP, PaddleOCR, and SoMark. For the deeper backend story, see [/08-deepdoc.md](08-deepdoc.md) and the PDF parser selection guide at [/docs/guides/dataset/select_pdf_parser.md](docs/guides/dataset/select_pdf_parser.md).

### Layout and structure-driven templates

Some genres carry enough native structure that a generic token budget throws away useful boundaries. `rag/app/paper.py` treats papers as sectioned arguments with abstracts, sections, and tables. `rag/app/book.py` leans on heading hierarchy and outline cleanup so long-form prose stays navigable. `rag/app/manual.py` and `rag/app/laws.py` both build trees from bullets and clauses, but they aim at different documents: manuals preserve task-oriented steps and page images, while laws preserve legal clause structure.

`rag/app/presentation.py` follows the page instead of the paragraph. Slide decks already encode their rhythm in the page break, so the chunker keeps one page-centered chunk and preserves page imagery when it matters. That choice gives retrieval a cleaner boundary than a synthetic split can provide.

### Record-shaped templates

`rag/app/qa.py`, `rag/app/table.py`, and `rag/app/resume.py` handle records rather than flowing prose. Their core idea stays simple: one record should become one chunk, with headers and fields preserved so search can match the record as a unit and still recover the important columns or labels.

`rag/app/resume.py` goes further than the others. It uses line-indexed extraction and parallel structured LLM passes, which gives it richer fields for work history, education, and projects. That extra work buys more accurate retrieval for dense personal profiles where a plain-text merge would lose the shape of the résumé.

### Whole-document and special-purpose templates

`rag/app/one.py`, `rag/app/picture.py`, `rag/app/audio.py`, `rag/app/email.py`, and `rag/app/tag.py` cover the degenerate or media-specific cases. `one.py` keeps the whole document together when splitting adds little value. `picture.py` turns images into searchable chunks with OCR and vision model help. `audio.py` transcribes speech before chunking. `email.py` combines headers, body, and attachments, and sends attachments through the naive path. `tag.py` treats a dataset as a controlled tag vocabulary and stores content-to-tag examples instead of ordinary prose.

## The shared machinery underneath

The zoo stands on one helper layer in `rag/nlp/__init__.py` and `rag/nlp/rag_tokenizer.py`. That layer owns the merge logic and the tokenization rules that every template depends on: `naive_merge`, `naive_merge_with_images`, `naive_merge_docx`, `hierarchical_merge`, `tree_merge`, `tokenize`, `tokenize_chunks`, `tokenize_table`, `add_positions`, and `attach_media_context`.

That shared layer solves two problems at once. First, it keeps the raw text close to the normalized token forms so full-text search can rank the original wording without re-parsing the source every time. Second, it lets layout-heavy chunks carry page and position metadata, and it lets media chunks borrow nearby text so images and tables do not land in the index as context-free islands. `rag/nlp/rag_tokenizer.py` wraps `infinity.rag_tokenizer` and preserves existing behavior unless `DOC_ENGINE_INFINITY` turns on the alternate engine.

## What a chunk is physically

A chunk is not just a text string. It usually carries the content that retrieval ranks, the tokenized content that indexing consumes, an embedding vector for semantic search, keywords for enrichment, source positions for traceability, and image or media markers when the source contains non-text material. The executor in `rag/svr/task_executor.py` stamps IDs, writes chunk rows into the document store, and stores image bytes in MinIO-backed storage before the chunk enters retrieval.

That storage contract matters because it keeps user-visible behavior stable even when the upstream template changes. The indexing layer sees a shaped chunk with text, metadata, and optional media, not a parser-specific artifact. For the storage and search boundary, see [/07-the-doc-engine-abstraction.md](07-the-doc-engine-abstraction.md).

## A dated note on the next path

As of July 2026, the parent-child strategy already handles the tension between recall and local context by keeping a parent chunk alongside narrower children. See [/docs/guides/dataset/configure_child_chunking_strategy.md](docs/guides/dataset/configure_child_chunking_strategy.md) for the user-facing version of that idea.

As of July 2026, `rag/flow/` also points past the fixed template zoo. `rag/flow/pipeline.py`, `rag/flow/chunker/token_chunker.py`, and `rag/flow/extractor/extractor.py` let parser, chunker, and extractor components compose as a graph, so the ingestion path can express more than a fixed template choice while still reusing the same chunk contract.

## Where to look in the code

- `rag/svr/task_executor.py` — dispatcher, `FACTORY`, persistence, and indexing.
- `rag/app/naive.py` — default token-budget chunking and PDF backend selection.
- `rag/app/paper.py`, `rag/app/book.py`, `rag/app/manual.py`, `rag/app/laws.py`, `rag/app/presentation.py` — structure-aware templates.
- `rag/app/qa.py`, `rag/app/table.py`, `rag/app/resume.py`, `rag/app/one.py`, `rag/app/picture.py`, `rag/app/audio.py`, `rag/app/email.py`, `rag/app/tag.py` — record-shaped and special cases.
- `rag/nlp/__init__.py` and `rag/nlp/rag_tokenizer.py` — merge helpers, positions, media context, and tokenization wrapper.
- `rag/flow/pipeline.py`, `rag/flow/chunker/token_chunker.py`, `rag/flow/extractor/extractor.py` — graph-based ingestion and extraction.