# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup & running

This repo currently contains one project: `vectorless-rag-pageindex/`, a single Jupyter notebook demo — there's no build step, lint config, or test suite.

```bash
cd vectorless-rag-pageindex
pip install -r requirements.txt
cp .env.example .env   # then fill in OPENAI_API_KEY
jupyter notebook vectorless_rag_pageindex.ipynb
```

The notebook asserts `OPENAI_API_KEY` is set (loaded via `python-dotenv`) and will fail immediately in cell 2 if it isn't.

## Architecture

`vectorless_rag_pageindex.ipynb` is a linear, cell-by-cell walkthrough (not a package/library) comparing **vectorless RAG** (via the `pageindex` package's `PageIndexLocalClient`) against traditional embedding-based RAG, on the same source document and question:

1. **Indexing** — `PageIndexLocalClient.submit_document()` parses a PDF into a hierarchical tree of titles, page ranges, and per-section summaries via an LLM call (through LiteLLM, using `OPENAI_API_KEY`). This runs fully local/self-hosted (`storage_path=".pageindex"`) — no PageIndex account or API key required.
2. **Manual reasoning-based retrieval** — the tree's titles+summaries only (via `client.get_document_structure`, never full text) are flattened with the in-notebook `flatten_titles_and_summaries`/`find_node` helpers and sent to an LLM to pick relevant `node_id`s; only those nodes' text is then read to answer the question.
3. **Agentic multi-hop retrieval** — `client.openai_agent_config(doc_id=...)` wires PageIndex's ready-made tool contracts (`browse_documents`, `get_document_structure`, `get_page_content`, ...) into an `openai-agents` `Agent`/`Runner`, letting the agent decide on its own which sections to open, potentially across several hops.
4. **Vector RAG baseline** (optional, final code cell) — naive fixed-size-word chunking of the same tree's text + `text-embedding-3-small` embeddings + cosine similarity, for a side-by-side comparison against the tree-search result.

Caching: the built document's `doc_id` is persisted to `data/sample-paper.doc_id` so reruns skip re-indexing (the most expensive, LLM-driven step). `data/` and `.pageindex/` are git-ignored local artifacts, not source — delete them to force a full rebuild.

For production use beyond this local demo, PageIndex also offers a hosted `PageIndexCloudClient` and an MCP server that accept the same tool contracts/agent config.
