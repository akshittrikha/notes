## Overview
A local semantic search + Q&A system over your Obsidian notes. Ask a question, get an answer sourced from things you've already written.

## Stack
- **Embeddings**: OpenAI `text-embedding-3-small` or a local model (Ollama + nomic-embed)
- **Vector DB**: Chroma (local) or pgvector (Postgres)
- **Retrieval + generation**: Claude API with retrieved chunks as context
- **Interface**: CLI or a local web UI (FastAPI + simple HTML)

## What you'll learn
- Full RAG pipeline: chunking, embedding, vector search, prompt construction
- Vector database basics (indexing, cosine similarity, metadata filtering)
- Tradeoffs in chunking strategies (fixed-size vs. sentence vs. paragraph)

## Core features
- [ ] Ingest all `.md` files from vault into vector DB
- [ ] Chunk notes by paragraph with overlap
- [ ] Embed chunks and store with source metadata
- [ ] Query interface: input a question, retrieve top-k chunks, generate answer
- [ ] Show source note links alongside answers
- [ ] Incremental re-index on vault changes (watch for file changes)

## Notes
- Start with Chroma — zero infra, runs in-process
- Obsidian links (`[[Note]]`) can be preserved as metadata for richer retrieval
- Good benchmark: can it answer "what did I write about X last month?" accurately
