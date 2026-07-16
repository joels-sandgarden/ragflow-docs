# The Chunking Template Zoo

RAGFlow keeps more than one chunking template because retrieval works best when the chunk boundary matches the document genre. A legal code, a slide deck, a resume, and an email thread each lose different kinds of meaning when a single splitter treats them the same way.

The dispatcher in `rag/svr/task_executor.py` forms the seam. `FACTORY` maps each `parser_id` to a module in `rag/app/`, and `build_chunks()` loads the file, merges parser configuration, and hands the task to `chunk()`. That split matters because the template zoo exists to preserve different kinds of structure, not to offer interchangeable ways of saying the same thing.

## The shared chunk contract

Every template returns chunk-shaped dictionaries, but each one emphasizes a different layer of the same record. `rag/app/naive.py` shows the plain version: it keeps the raw text in `content_with_weight` and adds tokenized search forms in `content_ltks` and `content_sm_ltks`. `rag/app/paper.py` and `rag/app/presentation.py` add page order, position fields, and page images when the document layout carries meaning.

That common shape gives the rest of the system a stable handoff. The retrieval layer can score the raw text, the search layer can work from normalized tokens, and the display layer can recover position or media context when the source demands it.

## The strategies in the zoo

### General and token budget

`rag/app/naive.py`, `rag/app/book.py`, `rag/app/manual.py`, and `rag/app/laws.py` aim for broad coverage. They merge text into token bounded chunks, keep sentence flow intact when possible, and let document structure steer the merge when headings or legal sections carry real meaning. This family works well when the source reads like prose with an internal spine, but it can blur local detail if the source mixes long narrative passages with dense reference material.

### Layout and structure driven

`rag/app/paper.py` and `rag/app/presentation.py` treat page order, headings, tables, and thumbnails as first class signals. The paper path protects the abstract, authorship, and section flow; the presentation path keeps each page as a chunk and preserves the page image. This approach keeps citations, page context, and visual order close to the text, even when the chunk size stops looking uniform.

### Record shaped

`rag/app/qa.py`, `rag/app/table.py`, `rag/app/resume.py`, and `rag/app/email.py` build records instead of prose blocks. A question pair, a spreadsheet row, a resume field bundle, or an email body with attachments already has an internal shape, so the chunker should preserve that shape instead of flattening it into a paragraph. The tradeoff is clear: these chunks answer targeted queries well, but they do not read like natural narrative excerpts.

### Whole document and special cases

`rag/app/one.py` keeps the original order and emits a single chunk per file or page set when the document boundary matters more than internal splitting. `rag/app/picture.py` and `rag/app/audio.py` turn media into text bearing chunks through OCR, vision, or transcription, which makes non textual sources searchable without pretending that the text came from a plain document. `rag/app/tag.py` sits apart from the rest of the zoo: it produces label bearing records for tag workflows, not a normal retrieval corpus.

## The shared machinery underneath

The merge and tokenization layer in `rag/nlp/__init__.py` and `rag/nlp/rag_tokenizer.py` gives the zoo its common language. `tokenize()` writes the raw text and its normalized token forms, while helpers such as `naive_merge`, `naive_merge_with_images`, `attach_media_context`, and `add_positions` keep source order, page geometry, and media context available when a parser needs them.

This dual form matters. Raw text keeps retrieval, embedding, and display faithful to the source. Tokenized forms give normalization, search, and keyword extraction a shared representation that downstream components can compare, cache, and rank.

## What a chunk becomes

The chunk does not stop as a parser output. `rag/svr/task_executor.py` adds document identity, timestamps, and knowledge base links, then stores the chunk in the document engine payload that downstream indexing consumes. `content_with_weight` becomes the retrieval facing body, token fields support search, and optional vector, keyword, question, and metadata fields extend the record when the pipeline asks for more than text.

The later ingestion path in `rag/flow/pipeline.py` makes that same handoff more explicit. It normalizes the upstream output, adds document and knowledge base fields, assigns positions, and prepares the final payload before indexing. That shape belongs in `./07-the-doc-engine-abstraction.md`; this chapter only needs to show that the chunk record eventually turns into a document engine document, not a temporary parser artifact.

> Note, July 2026: parent child chunking remains a separate layer that sits on top of these templates. The user level walkthrough lives in [/docs/guides/dataset/configure_child_chunking_strategy.md](/docs/guides/dataset/configure_child_chunking_strategy.md).

> Note, July 2026: the newer ingestion path in `rag/flow/pipeline.py` and `rag/flow/chunker/token_chunker.py` turns the same job into composable graph components. Fixed templates still matter, but the pipeline direction generalizes chunking into reusable nodes.

## Where to look in the code

- `rag/svr/task_executor.py`: `FACTORY` and `build_chunks()` select the parser template and start ingestion.
- `rag/app/naive.py`: the general token budget path and the media aware merge helpers live here.
- `rag/app/paper.py` and `rag/app/presentation.py`: these files show structure preserving and page aware chunking.
- `rag/app/table.py` and `rag/app/resume.py`: these files show record shaped chunk construction.
- `rag/nlp/__init__.py` and `rag/nlp/rag_tokenizer.py`: these files hold the shared tokenization and context helpers.
- `rag/flow/pipeline.py` and `rag/flow/chunker/token_chunker.py`: these files show the composable ingestion direction.