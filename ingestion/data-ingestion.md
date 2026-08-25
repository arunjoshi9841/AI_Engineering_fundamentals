# Data Ingestion

Retrieval quality starts before a user writes a query. Data ingestion turns source material into reliable, searchable chunks with enough context and metadata to retrieve them safely.

## The Problem

Knowledge usually arrives as files, web pages, database records, and exports. Those sources are designed for people or operational systems, not for retrieval.

A 200-page PDF may contain headings, tables, scanned pages, footnotes, and repeated page headers. A website may contain navigation, boilerplate, and dynamic content. Embedding either source as one large blob makes precise retrieval difficult. Splitting it carelessly can destroy the context that gives a sentence its meaning.

Ingestion creates a stable path from source data to searchable derived data.

## Mental Model

Treat ingestion as a small data pipeline. The source remains authoritative. The resulting chunks and indexes are rebuildable representations for search.

```text
Source
  ↓
Extract
  ↓
Normalize
  ↓
Chunk
  ↓
Enrich
  ↓
Embed
  ↓
Index
```

Each step should produce inspectable output. When retrieval fails, you need to determine whether the source was missing, extraction was wrong, a chunk boundary was poor, or the search index ranked the wrong content.

## How It Works

1. **Discover the source.** Read content from an approved file store, CMS, database, or website and record a stable source ID.
2. **Extract content.** Convert each source format into text and structured elements such as headings, paragraphs, tables, and image captions where available.
3. **Normalize it.** Clean encoding, remove irrelevant boilerplate, preserve useful structure, and retain the link to the original source.
4. **Chunk it.** Split the normalized content into units that can be retrieved precisely while retaining enough local context.
5. **Enrich each chunk.** Attach metadata such as title, section path, page number, language, permissions, and source URL.
6. **Embed and index it.** Generate an embedding for each chunk and write the vector, searchable text, metadata, and IDs to the retrieval stores.

```text
PDF or web page
  ↓ parse
Normalized document + structure
  ↓ chunk
Chunk text + source metadata
  ↓ embed
Vector + chunk record in retrieval indexes
```

## Important Concepts

### Documents Are Not Chunks

A document is the source object: a PDF, web page, database record, or ticket. A chunk is the smaller retrieval unit derived from that document.

One document commonly produces many chunks. Each chunk needs a stable relationship back to its parent document so the application can show a citation, fetch broader context, and later update or delete the derived data.

```text
employee-handbook.pdf
├── chunk: vacation policy
├── chunk: parental leave policy
└── chunk: expense reimbursement
```

Do not make an entire long document one chunk simply because it has one source URL. Retrieval should be able to select the relevant passage, not return every unrelated page.

### Extraction and Normalization

Extraction reads the actual content from a format. The difficulty depends on the source:

- **PDFs:** A text PDF may have columns or a broken reading order. A scanned PDF needs OCR. Tables and page references can be lost.
- **Websites:** Keep meaningful page content, headings, and canonical URLs. Remove navigation, cookie banners, and repeated footer text.
- **Tables:** Preserve headers with the rows they describe. A value without its row and column labels is often meaningless.
- **Images:** Preserve captions and surrounding text. Use OCR or multimodal extraction only when the image itself contains material the system must retrieve.

Normalization makes the extracted data consistent without discarding useful evidence. Typical work includes fixing encoding, collapsing duplicate whitespace, converting markup into a canonical form, and preserving heading hierarchy. Keep the raw source or a recoverable reference to it. Normalized text is a derived representation, not the original record.

### Chunking

Chunking chooses the retrieval unit. Start with natural structure: headings, paragraphs, list items, and table boundaries. Then enforce a maximum token size appropriate for the embedding model and later LLM context budget.

Chunk size has a direct tradeoff:

- Small chunks improve retrieval precision but can lose context and create many vectors.
- Large chunks preserve context but may match a broad topic while hiding the answer inside unrelated text.

Overlap repeats a small boundary region in adjacent chunks. It can preserve a sentence or idea that crosses a split, but it increases storage, embedding cost, and duplicate search results. Use it only when your content needs it.

There is no universal best chunk size or overlap. Start with a simple structural strategy, inspect real chunks, and evaluate against representative questions before adding semantic or hierarchical chunking.

### Metadata and Enrichment

Metadata is the structured information attached to a chunk. It supports filtering, citations, debugging, and access control.

A useful chunk record might contain:

```text
chunk_id
document_id
source_url
title
section_path
page_number
tenant_id
visibility
language
content_hash
```

Do not put metadata only in the embedding text. Store filterable fields separately, then also add selected context such as a title or section path to the text when it improves retrieval meaning.

### Embedding, Indexing, and Deduplication

After chunking, create embeddings with the selected embedding model and add the chunk to the sparse and vector indexes. Store the chunk text and metadata alongside the index IDs or in a reliable content store keyed by those IDs.

Deduplication prevents identical material from crowding out other results. An exact content hash is a practical first defense. For example, if a policy PDF is uploaded twice under different filenames, identical chunks should not appear twice in search results. Near-duplicate detection is possible later, but it needs careful evaluation because similar-looking documents may have important differences.

Record the embedding model and chunking version with the derived data. These details make later reindexing explainable.

## Where It Fits

Ingestion prepares the retrieval layer. It is separate from query-time retrieval and from the source system that owns the original content.

```text
CMS / database / files
     ↓
Ingestion pipeline
     ↓
Chunks + metadata + embeddings
     ↓
Sparse and vector indexes
     ↓
Query-time retrieval
```

This separation lets you rebuild indexes without changing the source of truth.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Structure-aware parsing | Better boundaries and citations | More format-specific handling |
| Smaller chunks | Precise retrieval | Less context and more index entries |
| Larger chunks | Broader local context | Lower precision and more prompt waste |
| Overlap | Continuity across boundaries | Duplicate content, storage, and cost |
| Rich metadata | Filtering, citations, and debugging | Schema design and maintenance |
| Aggressive deduplication | Less result repetition | Risk of collapsing meaningful variants |

## Failure Modes

- **Bad extraction:** OCR errors, lost table headers, or scrambled PDF columns make good retrieval impossible.
- **Boilerplate pollution:** Navigation and repeated legal text become common high-ranking chunks.
- **Broken context:** A chunk contains a conclusion but not the heading or condition that qualifies it.
- **Missing metadata:** The system cannot cite the source, filter by tenant, or remove stale content later.
- **Duplicate results:** Repeated exports or mirrored pages dominate retrieval.
- **Untraceable derived data:** A result cannot be mapped back to a source document or ingestion version.

## Example

A company ingests its benefits handbook as a PDF. The parser extracts headings, paragraphs, tables, and page numbers. The pipeline removes repeated page headers, splits at section boundaries, and preserves a small overlap only where a paragraph continues across a boundary.

Each chunk gets the handbook URL, section path, page number, audience, and a content hash before it is embedded and indexed. A search for "how much time off for a new child" can retrieve the parental-leave section and show the original page, instead of returning the whole handbook.

## Interview Takeaways

- Ingestion converts source documents into derived chunks, metadata, and index entries. It does not replace the source of truth.
- Parsing and chunking quality place an upper bound on retrieval quality.
- Chunk size and overlap trade precision, context, cost, and duplicate results.
- Metadata is necessary for filtering, citations, debugging, and access control.
- Stable document and chunk identities make reindexing and deletion possible.

## Next

Next: Editable Content and Reindexing. It explains how to keep these derived indexes correct when source content changes.

## References

- [Apache Tika: content detection and extraction](https://tika.apache.org/)
- [Unstructured: chunking documentation](https://docs.unstructured.io/open-source/core-functionality/chunking)
- [Amazon Bedrock: how content chunking works for knowledge bases](https://docs.aws.amazon.com/en_en/bedrock/latest/userguide/kb-chunking.html)
- [Amazon Bedrock: metadata in a data source](https://docs.aws.amazon.com/en_en/bedrock/latest/userguide/kb-metadata.html)
