# About this site

This site presents a concept-level field guide to the RAGFlow codebase. It explains why the major subsystems exist, which ideas organize them, and how the pieces fit together so an engineer can build a mental model without reading the code end to end. The guide frames the rest of the series and keeps the focus on concepts and relationships rather than setup steps or API reference.

The intended reader is an engineer adopting, operating, extending, or contributing to RAGFlow. The guide assumes comfort with code and infrastructure, but it keeps the focus on the shape of the system rather than on individual commands or screens. That gives the reader enough context to use the code maps in the other chapters without treating this page as a tutorial.

Doc Holiday (https://doc.holiday) wrote this guide directly from the RAGFlow source repository. The pages reflect the repository at the snapshot recorded below and use the code as the primary source of truth. The writing follows the source structure, so each chapter names a subsystem, explains the idea that holds it together, and shows how it connects to neighboring parts of the system. Readers can use the overview first and then move toward the chapter that matches the work at hand.

Repository snapshot: [GENERATED_FROM: commit SHORT_SHA, DATE — the operator will replace this placeholder; include it verbatim if you cannot determine it]

The official documentation at https://ragflow.io/docs/dev/ remains the authoritative source for quickstarts, UI guidance, component and API reference, and operations. This field guide complements that material by describing the architecture and the path a request follows through the system. It does not repeat procedural material or compete with the product documentation; it explains the mental model that makes that material easier to use. In that role, the site bridges a first pass through the product and a deeper reading of the source tree.

The repository changes actively. When the guide mentions migration work or transitional behavior, those notes describe the state of the code at the time of writing. They do not define the main subject of the guide. Those dated observations keep the guide honest in a live repository, but they stay secondary to the architecture story.

This guide names the load-bearing paths and the decisions that shape them, then leaves detailed operation to the official material. Each chapter starts with the subsystem's purpose and ends with a short map of the code paths that carry that idea. The series map below follows the same progression from the broad system view to query flow, ingestion, chunking, embeddings, graph retrieval, canvas orchestration, document engines, and DeepDoc. That order lets a reader move from context to detail without losing the thread. It also keeps the closing page concise, so the table of contents carries the navigation load.

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