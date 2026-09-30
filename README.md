# Week 1 Enterprise RAG - Production-Structured Notebook Pack

This project turns the Week 1 learning sequence into small, runnable notebooks. Each notebook is self-contained, reads from `data/nexabank_data`, and mirrors the production-style behavior in `src/week1_rag` without importing it.

## Architecture

```text
OFFLINE INGESTION
NexaBank Markdown policies + CSV metadata
    -> LangChain loaders
    -> RecursiveCharacterTextSplitter
    -> OpenAI text-embedding-3-small (768 dimensions)
    -> MongoDB Vector Search

ONLINE RAG
Question
    -> same embedding space used at ingestion
    -> MongoDB Vector Search retriever
    -> top-k documents
    -> LangChain prompt + ChatOpenAI (GPT-6 Luna)
    -> grounded answer + source documents
```

## Notebook order

1. `00_setup_and_architecture.ipynb`
2. `01_document_loading.ipynb`
3. `02_chunking_and_metadata.ipynb`
4. `03_embedding_provider.ipynb`
5. `04_mongodb_vector_ingestion.ipynb`
6. `05_semantic_retrieval.ipynb`
7. `06_parent_document_retrieval.ipynb`
8. `07_rag_generation.ipynb`
9. `08_production_checks.ipynb`

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
# Windows: .venv\Scripts\activate
pip install -e .
cp .env.example .env
jupyter lab
```

The notebooks always run their local dataset checks. External operations are
opt-in so opening or executing a notebook does not write to MongoDB or call a
model unexpectedly:

```bash
RUN_EMBEDDING_CHECK=1   # notebook 03
RUN_VECTOR_INGESTION=1  # notebook 04
RUN_VECTOR_BOOTSTRAP=1 RUN_RETRIEVAL=1  # notebook 05, new collection
RUN_PARENT_INGESTION=1 RUN_PARENT_RETRIEVAL=1  # notebook 06
RUN_VECTOR_BOOTSTRAP=1 RUN_RAG=1        # notebook 07, new collection
RUN_VECTOR_BOOTSTRAP=1 RUN_PRODUCTION_CHECKS=1 # notebook 08, new collection
```

Add `OPENAI_API_KEY` to `.env` before enabling any model-backed cell. The key is
never placed in notebook source or printed by the notebooks.

## OpenAI model choices

- `gpt-6-luna` is the default answer model because this workload is focused,
  high-volume policy question answering. Generation uses no reasoning and is
  capped at 400 output tokens.
- `text-embedding-3-small` is the default embedding model. Vectors are shortened
  from 1,536 to 768 dimensions to reduce MongoDB storage and index size.
- Ingestion batches embedding requests and skips sources whose checksum and
  chunk count have not changed.
- If authorized retrieval returns no context, the application answers without
  making an LLM request.

The embedding provider, model, and dimensions are stored in MongoDB. Never mix
different embedding spaces in one collection. Use a fresh collection such as
`policy_chunks_openai_768` when migrating from the previous embedding setup.

## Production decisions demonstrated

- credentials come from environment variables, not notebook source;
- each notebook can run from a fresh kernel without executing an earlier notebook;
- notebook-local code mirrors `src/week1_rag` while loading the corpus directly from `data/nexabank_data`;
- source and chunk IDs are deterministic;
- the canonical Markdown policies are indexed without their duplicate PDF exports;
- policy metadata is attached to every LangChain `Document` and chunk;
- `allowed_groups` is enforced as a MongoDB Vector Search pre-filter before retrieval;
- unchanged sources are not re-embedded on repeated ingestion runs;
- changed sources are replaced instead of blindly duplicated;
- the embedding provider, model, and vector dimensions are persisted and checked;
- MongoDB's LangChain integration is used rather than a custom vector-store wrapper;
- MongoDB's parent-document retriever is used rather than reimplementing that pattern;
- the project uses LangChain's dedicated OpenAI chat and embedding integrations.

## Official references used

- MongoDB LangChain integration: https://www.mongodb.com/docs/atlas/ai-integrations/langchain/
- MongoDB Vector Search filtering: https://www.mongodb.com/docs/vector-search/query/aggregation-stages/vector-search-stage/
- MongoDB parent document retrieval: https://www.mongodb.com/docs/atlas/ai-integrations/langchain/parent-document-retrieval/
- OpenAI model catalog: https://developers.openai.com/api/docs/models
- OpenAI embeddings guide: https://developers.openai.com/api/docs/guides/embeddings
