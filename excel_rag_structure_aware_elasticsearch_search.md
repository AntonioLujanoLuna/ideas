# Structure-Aware RAG over Excel Workbooks

**Status:** Design proposal · **Updated:** 2026-10-09  
**Scope:** Excel ingestion, Elasticsearch indexing, and a stateless FastAPI retrieval endpoint.  
**Constraint:** Elasticsearch is the **only persistent search/vector/metadata store**. Reuse the existing BM25, embeddings, and ColBERT retrieval infrastructure.

## 1. Idea summary

Treat an Excel workbook like a codebase: worksheets resemble source files, tables and named ranges resemble symbols, and formulas create a dependency graph. **Index both semantic content and workbook structure in Elasticsearch**, preserving coordinates, headings, formulas, provenance, and cross-sheet references. A FastAPI endpoint searches the semantic index and, optionally, expands related structural nodes with bounded Elasticsearch lookups.

Do **not** flatten each worksheet to Markdown and chunk it blindly. Also do **not** introduce a LangGraph agent, multi-turn search, relevance evaluation, SQL engine, graph database, or another persistent datastore. Answer synthesis and follow-up reasoning belong to the caller.

**Design principles**
1. **Structure first:** preserve workbook -> sheet -> region/table -> formula/cell hierarchy.
2. **Embed meaningful regions:** workbook/sheet descriptions, tables, column schemas, row groups, and selected formula summaries, not every numeric cell.
3. **Resolve relationships at ingestion:** formula precedents, named ranges, cross-sheet dependencies, and candidate shared keys.
4. **Keep results exact:** return values, formulas, A1 coordinates, and file/version identifiers alongside search snippets.
5. **Bound retrieval:** reference traversal has depth, node, payload-size, and timeout limits.
6. **Fail explicitly:** dynamic or external references are recorded as unresolved instead of inventing graph edges.

## 2. System architecture

```text
Excel (.xlsx / .xlsm)
         |
  Ingestion worker
  openpyxl + region detection + formula parser
         |
  Canonical workbook representation (in memory)
  hierarchy / ranges / tables / values / dependencies
         |
  Existing embedding pipeline + Elasticsearch bulk indexing
         |
         +--------------------------+
         | Elasticsearch            |
         | excel_chunks             | <- BM25 / dense vectors / ColBERT
         | excel_structure          | <- metadata / values / edges
         +--------------------------+
                        ^
                        |
               Stateless FastAPI
          POST /api/v1/search/excel
                        |
         hits + coordinates + optional
         related nodes / unresolved refs
```

The canonical representation exists only during ingestion. Elasticsearch is the durable source for retrieval and reference resolution. No extra database is required.

## 3. Ingestion plan

### 3.1 Read and normalize

- Use **openpyxl** with `data_only=False` for formulas and a second `data_only=True` read for last-saved cached values. A cached value is **not** a verified, freshly computed result.
- Inventory worksheets, named ranges, Excel tables, merged cells, hidden rows/sheets, comments, hyperlinks, and chart series where supported.
- Detect separate tables/regions within one sheet using populated-cell boundaries, blank separators, merged headings, table objects, formatting, and header heuristics. Do not equate a worksheet with a single table.
- Annotate every region with its worksheet, inclusive A1 rectangle, header context, columns, likely data types, units, titles, and nearby notes.
- Preserve typed values, display formats, formula text, cached results, and provenance. Do not execute macros, refresh links, or evaluate formulas during ingestion.
- Separate content for embedding from the underlying exact values: region descriptions and contextualized row-group text are searchable; exact cell/range payloads are retrievable separately.

### 3.2 Parse relationships

Store a *hierarchy* (parent/children) and a *dependency graph* (typed directed edges):

| Relationship | Example | Handling |
|---|---|---|
| Hierarchy | workbook -> sheet -> revenue table | Explicit parent and child IDs |
| Formula precedent | `Forecast!F18` uses `Assumptions!C7` | Parse and resolve to target node/range |
| Cross-sheet range | `SUM(Actuals!D2:D500)` | Edge to range/region, not 499 separate cells |
| Named range | `GrowthRate` -> `Assumptions!C7` | Resolve with workbook/sheet scope |
| Structured table reference | `Sales[Revenue]` | Resolve against table and column metadata |
| Implicit join candidate | matching `customer_id` columns | Record as **unverified**, never as a proven dependency |

Canonicalize addresses, including quoted sheet names and absolute/relative references. Group repeated formulas into patterns when useful. Persist unresolved `INDIRECT`, dynamic `OFFSET`, external links, unsupported dynamic arrays, and similar cases with the reason for failure. Some references require a calculation engine and cannot be statically determined.

**Key optimization:** edges point to stable nodes or rectangular ranges. For a reference covering a large area, index the rectangle and resolve overlapping tables/regions through range queries, rather than creating one edge per cell.

### 3.3 Chunking

Use multiple granularity levels:

- **Workbook/sheet:** title, purpose, worksheet inventory, region summaries.
- **Table/region:** descriptive title, context, column headers, units, range, formula summaries.
- **Column schema:** name, type, aliases, examples, distinct/key statistics if inexpensive.
- **Row groups:** a bounded number of rows with inherited table headings and column labels; retain row coordinates.
- **Formula summaries:** output label + formula + referenced inputs (optionally also indexed as metadata).

Do not fragment one logical table simply at token boundaries. For very large datasets, keep bounded row-group payload documents without necessarily embedding every row. If exact exhaustive aggregates are required later, that is a separate computation capability, not a promise of this search endpoint.

## 4. Elasticsearch data model

Use **two indices**, sharing `workbook_id`, `version`, `sheet_id`, and `node_id`. Store semantic content in `excel_chunks` and structural nodes/edges in `excel_structure`.

### 4.1 `excel_chunks`

Illustrative document:

```json
{
  "id": "wb42:v3:chunk:revenue_forecast",
  "workbook_id": "wb42",
  "version": 3,
  "node_id": "wb42:v3:region:forecast_revenue",
  "sheet_id": "wb42:v3:sheet:forecast",
  "sheet_name": "Forecast",
  "a1_range": "A12:F25",
  "chunk_type": "table",
  "title": "Revenue forecast",
  "content": "Revenue projections by year, linked to historical actuals and growth assumptions.",
  "headers": ["Metric", "2027", "2028", "2029", "2030"],
  "embedding": [0.12, 0.45, 0.78],
  "acl_scope": ["finance-team"]
}
```

The embedding is illustrative; its real dimensionality and mapping follow the existing retrieval stack. Retain any current ColBERT representation if already supported. Standard BM25 applies to titles, sheet names, headers, and content.

### 4.2 `excel_structure`

Illustrative formula node:

```json
{
  "node_id": "wb42:v3:formula:forecast!f18",
  "workbook_id": "wb42",
  "version": 3,
  "node_type": "formula",
  "sheet_id": "wb42:v3:sheet:forecast",
  "sheet_name": "Forecast",
  "a1_range": "F18",
  "row_span": {"gte": 18, "lte": 18},
  "column_span": {"gte": 6, "lte": 6},
  "parent_id": "wb42:v3:region:forecast_revenue",
  "formula": "=SUM(Actuals!D2:D500)*(1+Assumptions!C7)",
  "cached_value": 1250000,
  "references": [
    {
      "target_node_id": "wb42:v3:range:actuals!d2:d500",
      "sheet_name": "Actuals",
      "a1_range": "D2:D500",
      "kind": "range"
    },
    {
      "target_node_id": "wb42:v3:cell:assumptions!c7",
      "sheet_name": "Assumptions",
      "a1_range": "C7",
      "kind": "cell"
    }
  ],
  "unresolved_references": []
}
```

Use actual Elasticsearch `integer_range` fields for `row_span` and `column_span`, plus keyword fields for exact identifiers and names. A target range should resolve to a persisted structural node or to a canonical, queryable range descriptor; never emit a dangling `target_node_id`. For range intersection, filter by **workbook, version, sheet**, then intersect both row and column spans; choose the appropriate table/range node rather than returning all overlapping nodes blindly.

For direct lookups, index small important cells/formulas individually; store large grids as bounded range/row-group nodes. Keep the content size manageable and avoid Elasticsearch's document-count explosion from cell-per-document indexing.

**IDs and versioning:** IDs must be deterministic within `(workbook_id, version)`. Persist an active-version manifest in Elasticsearch, index a replacement version before activating it, then garbage-collect older versions. Never expand references across versions. Apply access-control filters to primary searches **and** related-node lookups; `_mget` does not itself enforce document-level tenant ACLs.

## 5. FastAPI retrieval contract

Primary route: `POST /api/v1/search/excel`.

**Request**

```json
{
  "query": "How is projected revenue calculated?",
  "filters": {"workbook_ids": ["wb42"]},
  "top_k": 10,
  "include_structure": true,
  "expand_references": true,
  "reference_depth": 1,
  "max_related_nodes": 20
}
```

**Response (abbreviated)**

```json
{
  "hits": [
    {
      "chunk_id": "wb42:v3:chunk:revenue_forecast",
      "score": 0.92,
      "content": "Revenue projections by year...",
      "source": {
        "workbook_id": "wb42",
        "version": 3,
        "sheet": "Forecast",
        "a1_range": "A12:F25"
      },
      "node_id": "wb42:v3:region:forecast_revenue",
      "related_node_ids": ["wb42:v3:cell:assumptions!c7"]
    }
  ],
  "nodes": {
    "wb42:v3:cell:assumptions!c7": {
      "sheet": "Assumptions",
      "a1_range": "C7",
      "value": 0.05
    }
  },
  "unresolved_references": [],
  "truncated": false
}
```

Scores and values above are illustrative. The response is **retrieval evidence, not a generated answer**.

**Request processing**
1. Resolve authorized workbook IDs and active versions; enforce existing document ACLs.
2. Run the existing BM25/vector/ColBERT retrieval workflow against `excel_chunks` (reusing existing query embedding infrastructure).
3. Fetch matching nodes from `excel_structure` with batched queries/`_mget`, validating authorization and version.
4. If requested, breadth-first-expand references up to the depth/node/byte/time budgets, deduplicate IDs, and resolve range intersections in batches.
5. Return hits, exact coordinates, optional related nodes, unresolved references, and truncation metadata.

Avoid one Elasticsearch request per cell or edge. Additional optional endpoints such as `GET /api/v1/excel/{workbook_id}/structure` or `POST /api/v1/excel/range` can be added if clients need deterministic inspection independently of semantic search.

**Out of scope:** multi-turn orchestration, agent tool selection, answer generation, feedback-based relevance evaluation, executing arbitrary spreadsheets, SQL analytics, recalculating Excel formulas, and introducing new persistent stores.

## 6. Implementation phases

**Phase 1: MVP ingestion and search**
- Parse `.xlsx` and `.xlsm` without executing macros.
- Index sheets, tables/regions, contextualized chunks, headers, exact A1 coordinates, and cached values.
- Deliver stateless FastAPI hybrid search with source citations and ACL/version filters.
- Test worksheets containing multiple separated tables and merged headings.

**Phase 2: Cross-reference retrieval**
- Extract formula precedents and named ranges.
- Add structural nodes, range overlap queries, reference expansion, batching and cycle detection.
- Record unsupported/dynamic references explicitly.
- Support parent/child context and references crossing multiple worksheets.

**Phase 3: Production hardening**
- Active-version switching, deterministic reindexing, stale-version cleanup, bulk ingestion retries.
- Per-request depth/node/time/byte budgets, monitoring, large-workbook benchmarks.
- Harden parsing against zip bombs, external references, untrusted hyperlinks, malicious formulas, and oversized sheet dimensions.
- Optionally enrich embedded charts/visual elements if spreadsheet retrieval actually requires them.

## 7. Acceptance criteria

- A query can find content in **different tables on the same worksheet** and return precise A1 locations.
- A formula hit in one worksheet can return its **statically resolved references in another** without model reasoning.
- Retrieval never mixes workbook versions or crosses access scopes, including during dependency expansion.
- Large range dependencies remain compact; no default cell-per-document or cell-per-edge explosion.
- Search results include sufficient source provenance to open the correct workbook, sheet, and range.
- Performance is measured independently for standard search and optional reference expansion (p50/p95, Elasticsearch requests, indexed docs per workbook, bytes returned).
- Complex/dynamic references and truncated expansions are clearly identified rather than silently omitted.

## 8. Implementation recommendation

Start with **openpyxl + your existing embedding/ColBERT pipeline + two Elasticsearch indices + one FastAPI endpoint**. Treat structural metadata the way your code RAG already treats symbols and dependencies. Concentrate engineering effort on **region detection, accurate reference resolution, and bounded result expansion**, rather than introducing another reasoning layer.

**Further reading:** [openpyxl documentation](https://openpyxl.readthedocs.io/) · [Elasticsearch range field types](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/range) · [Elasticsearch multi-get API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-mget).
