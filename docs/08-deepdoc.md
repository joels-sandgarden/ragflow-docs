# DeepDoc

DeepDoc is RAGFlow's document understanding layer. It turns page images and structured source files into ordered text, tables, and figure regions, then hands that material to chunking for retrieval. Layout carries meaning here: a title, a table cell, a caption, a header, or a figure region changes what retrieval can recover later.

## Series map

- [02 anatomy of ingestion](./02-anatomy-of-ingestion.md)
- [03 the chunking template zoo](./03-the-chunking-template-zoo.md)

## Why reading a PDF needs vision

PDFs preserve appearance more faithfully than structure. DeepDoc reads that appearance as a layout problem because a visually correct page can still hide the wrong reading order, broken table cells, repeated headers, or captions that drift away from the content they describe. If the parser drops those boundaries too early, chunking mixes unrelated text together and retrieval loses the signal that matters. The result matters downstream: the retriever does not recover meaning that the parser already erased.

## Two halves of `deepdoc/`

`deepdoc/parser/` handles the format side. It exports `PdfParser`, `DocxParser`, `EpubParser`, `ExcelParser`, `HtmlParser`, `JsonParser`, `MarkdownParser`, and `TxtParser`. `DocxParser` gives a useful contrast because it can read paragraphs, tables, images, and page breaks directly. `PdfParser` carries the special case because PDF often arrives as rendered pages, not as clean document objects.

`deepdoc/vision/` handles the model side. It exports `OCR`, `LayoutRecognizer`, `AscendLayoutRecognizer`, and `TableStructureRecognizer`. `OCR` finds text boxes and recognizes text inside them. `LayoutRecognizer` and `AscendLayoutRecognizer` classify page regions with the labels `Text`, `Title`, `Figure`, `Figure caption`, `Table`, `Table caption`, `Header`, `Footer`, `Reference`, and `Equation`. `TableStructureRecognizer` adds the table grid labels `table`, `table column`, `table row`, `table column header`, `table projected row header`, and `table spanning cell`.

## The PDF pipeline

`deepdoc/parser/pdf_parser.py` ties the pieces together. `RAGFlowPdfParser` and `Pdf` start with page images, run OCR, classify layout, send table regions through TSR, and then merge the text into a reading order that preserves position metadata. That position data matters for two reasons. First, the UI uses it for page highlighting and region cropping. Second, the figure and table paths use it to extract the original page regions instead of guessing from plain text alone.

The pipeline keeps the geometry instead of flattening it away. That choice adds complexity, but it keeps the page faithful to the source. Tables stay aligned with their cells, captions stay attached to the thing they describe, and figure regions remain available for later enrichment. The output can then move into the chunking step in [03 the chunking template zoo](./03-the-chunking-template-zoo.md), where DeepDoc's ordered sections and tables become tokenized chunks.

## Build versus integrate

RAGFlow owns DeepDoc as the default path, but `rag/app/naive.py` also routes datasets to other backends when the dataset config asks for them. The backend list currently includes `mineru`, `docling`, `opendataloader`, `tcadp parser`, `paddleocr`, `somark`, and `plaintext`, with `deepdoc` as the default branch. The app keeps the split simple: use DeepDoc when the default parser path fits, switch to another backend when the dataset needs a different extraction strategy, and then hand the result to the same merge and tokenization flow.

The shared wrapper layer in `rag/llm/cv_model.py` exists because several of those paths need the same computer vision and vision-language model interface. It lets image description and figure enrichment reuse one model abstraction instead of scattering provider specific code across parsers. As of July 2026, the backend list in `rag/app/naive.py` keeps growing. The architectural choice stays the same.

## What DeepDoc consumes and what it feeds

The ingestion worker and parser dispatch layer hand raw uploads into the parser boundary, then normalize the result into `schema.Page` records before chunking begins. See [02 anatomy of ingestion](./02-anatomy-of-ingestion.md) for that boundary.

DeepDoc then feeds the chunk templates in [03 the chunking template zoo](./03-the-chunking-template-zoo.md). The parser output gives chunking ordered text, tables, page positions, and, when relevant, the figure path from `deepdoc/parser/figure_parser.py`, which lets the system enrich images with vision descriptions instead of treating them as dead attachments.

## Where to look in the code

- `deepdoc/parser/pdf_parser.py` — the main PDF flow: OCR, layout, table structure, text merge, and position retention.
- `deepdoc/vision/ocr.py` and `deepdoc/vision/layout_recognizer.py` — text box detection, recognition, and page region classification.
- `deepdoc/vision/table_structure_recognizer.py` — TSR labels, table reconstruction, and caption handling.
- `deepdoc/parser/docx_parser.py` — the structured document path that contrasts with the PDF vision pipeline.
- `deepdoc/parser/figure_parser.py` — figure enrichment and image description through the vision model layer.
- `rag/app/naive.py`, `rag/llm/cv_model.py`, `internal/ingestion/component/parser.go`, and `internal/ingestion/component/parser_dispatch.go` — backend selection, shared CV wrappers, and the ingestion boundary.