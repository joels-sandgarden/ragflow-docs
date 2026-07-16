# About this site

This site presents a concept-level field guide to the RAGFlow codebase. It explains why the major subsystems exist, which ideas organize them, and how the pieces fit together so an engineer can build a mental model without reading the code end to end.

The intended reader is an engineer adopting, operating, extending, or contributing to RAGFlow. The guide assumes comfort with code and infrastructure, but it stays above the API reference layer and focuses on the structure of the system itself.

Doc Holiday (https://doc.holiday) wrote this guide directly from the RAGFlow source repository. The pages reflect the repository at the snapshot recorded below and use the code as the primary source of truth.

Repository snapshot: [GENERATED_FROM: commit SHORT_SHA, DATE — the operator will replace this placeholder; include it verbatim if you cannot determine it]

The official documentation at https://ragflow.io/docs/dev/ remains the authoritative source for quickstarts, UI guidance, component and API reference, and operations. This field guide complements that material by describing the architecture and the path a request follows through the system.

The repository changes actively. When the guide mentions migration work or transitional behavior, those notes describe the state of the code at the time of writing. They do not define the main subject of the guide.

The pages follow a simple reading order from the whole system to narrower subsystems, but each page also stands on its own as a short explanation of one part of the codebase.

Corrections or repository links: [CONTACT_OR_REPO_LINK placeholder for the operator]

## Series map

- [00 the big picture](./00-the-big-picture.md) — The overall shape of the system and the main subsystems that support it.
- [01 anatomy of a query](./01-anatomy-of-a-query.md) — How a chat question moves through retrieval, reranking, prompting, and citations.
- [02 anatomy of ingestion](./02-anatomy-of-ingestion.md) — How source material enters the system and becomes usable knowledge.
- [03 the chunking template zoo](./03-the-chunking-template-zoo.md) — How chunking templates shape documents before retrieval.
- [04 the embedding layer](./04-the-embedding-layer.md) — How embeddings are produced, stored, and matched.
- [05 graphrag](./05-graphrag.md) — How graph-based retrieval adds another route to relevant context.
- [06 the canvas orchestrator](./06-the-canvas-orchestrator.md) — How the canvas layer coordinates longer running work.
- [07 the doc engine abstraction](./07-the-doc-engine-abstraction.md) — How document engines present a common interface across backends.
- [08 deepdoc](./08-deepdoc.md) — How DeepDoc handles document parsing and extraction.