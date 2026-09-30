# NexaBank RAG Notebooks

A self-contained notebook project for learning an enterprise retrieval-augmented
generation (RAG) workflow with LangChain, OpenAI, and MongoDB Atlas Vector Search.
The included policies and evaluation questions are synthetic.

## Project contents

```text
data/nexabank_data/
├── fictional_policies/       # 12 canonical Markdown policies
├── pdf_exports/              # 3 derived PDF examples; not indexed by default
├── document_metadata.csv     # policy metadata and allowed groups
├── evaluation_questions.csv  # 60 retrieval/RAG questions
└── evaluation_questions.jsonl

notebooks/
├── 00_setup_and_architecture.ipynb
├── 01_document_loading.ipynb
└── 02_chunking_and_metadata.ipynb
```

Each notebook runs independently from a fresh kernel and reads directly from
`data/nexabank_data`.

## Notebook guide

| Notebook | Purpose |
|---|---|
| `00_setup_and_architecture.ipynb` | End-to-end architecture, local validation, optional OpenAI embeddings, MongoDB ingestion, retrieval, and RAG |
| `01_document_loading.ipynb` | Canonical Markdown loading, metadata enrichment, provenance, and schema validation |
| `02_chunking_and_metadata.ipynb` | LangChain recursive chunking, deterministic chunk IDs, and metadata preservation |

## Architecture

```text
Markdown policies + CSV metadata
    -> LangChain Documents
    -> RecursiveCharacterTextSplitter
    -> deterministic chunk IDs
    -> OpenAI text-embedding-3-small
    -> MongoDB Atlas Vector Search
    -> authorization-filtered retrieval
    -> OpenAI grounded answer
```

The Markdown policies are the canonical corpus. The PDFs are derived copies and
are excluded from normal ingestion to prevent duplicate retrieval results.

## Setup

Requires Python 3.11 or newer.

```bash
python -m venv .venv
source .venv/bin/activate

python -m pip install \
  jupyterlab python-dotenv pypdf pymongo \
  langchain langchain-community langchain-text-splitters \
  langchain-openai langchain-mongodb

cp .env.example .env
jupyter lab
```

On Windows, activate the environment with `.venv\Scripts\activate`.

## Configuration

Set these values in `.env` before enabling live operations:

```dotenv
OPENAI_API_KEY=...
MONGODB_URI=...
```

The supplied defaults use:

- `gpt-6-luna` for answer generation;
- `text-embedding-3-small` with 768 dimensions for embeddings; and
- `policy_chunks_openai_768` as the MongoDB vector collection.

Never commit `.env` or place credentials directly in notebook cells.

## Running live operations

Notebook 00 runs local loading and validation by default. External calls and
database writes are controlled by opt-in environment flags:

| Flag | Action |
|---|---|
| `RUN_EMBEDDING_CHECK=1` | Make one OpenAI embedding check |
| `RUN_MONGODB_CHECK=1` | Ping MongoDB and read stored embedding configuration |
| `RUN_VECTOR_INGESTION=1` | Create/update the vector index and embed changed policies |
| `RUN_RAG=1` | Retrieve authorized context and generate an answer |

After editing `.env`, restart the kernel or rerun the notebook's setup and
configuration cells.

## Safety and cost controls

- Retrieval applies `allowed_groups` as a MongoDB pre-filter before similarity
  ranking.
- Provider, embedding model, and vector dimensions are stored and checked before
  reuse. Use a new collection when changing the embedding space.
- Unchanged source files are not embedded again.
- Changed or incomplete sources are replaced rather than duplicated.
- Generation disables reasoning and caps output at 400 tokens by default.
- No authorized context means no chat-model request.

## Dataset notice

All organizations, policies, dates, owners, metrics, and rules in the NexaBank
dataset are fictional and intended only for training and demonstration. They are
not banking, regulatory, legal, or security requirements.
