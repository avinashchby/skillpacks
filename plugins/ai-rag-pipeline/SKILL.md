---
name: ai-rag-pipeline
description: >
  Designs and implements end-to-end Retrieval-Augmented Generation (RAG) pipelines:
  document loading, text chunking, embedding generation, vector store indexing, and
  retrieval chain construction. Supports Qdrant, Chroma, and Pinecone as vector stores.
  Covers chunking strategies (fixed, recursive, semantic), hybrid search (dense + sparse),
  reranking, and evaluation. Trigger phrases: "setup rag", "rag pipeline", "vector
  search", "document qa", "semantic search", "build rag", "retrieval augmented",
  "index documents", "search my documents", "qa over documents", "document search".
version: 1.0.0
---

# RAG Pipeline Builder

## When to Use

Activate this skill when the user:
- Wants to build a Q&A system over their own documents, PDFs, or knowledge base
- Says "setup rag", "rag pipeline", "semantic search", or "document qa"
- Needs to index a corpus and retrieve relevant chunks before generating an answer
- Wants to choose between or migrate between vector stores (Qdrant, Chroma, Pinecone)
- Asks about chunking strategies, embedding models, or retrieval quality

Do NOT activate for simple keyword search, SQL full-text search, or tasks where
the corpus is small enough to fit entirely in the LLM context window.

---

## Instructions

### Step 1 — Gather requirements

Confirm the following before generating any code:

1. **Document types** — PDF, Markdown, HTML, DOCX, plain text, database rows?
2. **Corpus size** — Number of documents and approximate total size (MB/GB)
3. **Vector store preference** — Qdrant (default), Chroma (local/simple), or Pinecone (managed)
4. **Embedding model** — OpenAI `text-embedding-3-small` (default), local (`nomic-embed-text`
   via Ollama), or Cohere
5. **Query type** — Pure semantic search, hybrid (semantic + keyword), or structured
   (filter by metadata + semantic)
6. **Framework** — LangChain (default for rapid prototyping), LlamaIndex, or custom
7. **Reranking** — Required for high-precision use cases? (adds latency, improves relevance)
8. **Language** — Python (default). TypeScript on request.

State all inferred assumptions explicitly.

### Step 2 — Design the pipeline architecture

Draw the pipeline as a component list before writing code:

```
[Documents]
    ↓ Loader (PyMuPDF / Unstructured / custom)
[Raw text + metadata]
    ↓ Chunker (recursive / semantic / sentence)
[Chunks with source metadata]
    ↓ Embedder (text-embedding-3-small / nomic-embed-text)
[Vectors + metadata]
    ↓ Vector Store (Qdrant / Chroma / Pinecone)
         ↕ retrieval
[Top-K chunks] → Reranker (optional) → [Reranked chunks]
    ↓ Context assembly
[Prompt: system + context + question]
    ↓ LLM (GPT-4o / Claude 3.5 / etc.)
[Answer with source citations]
```

### Step 3 — Choose and justify the chunking strategy

**Fixed-size chunking (baseline)**
Use when: documents are uniformly structured (logs, database exports)
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,        # 12% overlap to avoid splitting mid-sentence
    separators=["\n\n", "\n", ". ", " "],
)
```
Tradeoff: Fast and predictable, but splits mid-concept on prose documents.

**Recursive / document-aware (recommended default)**
Use when: mixed document types, prose-heavy content
```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[("##", "section"), ("###", "subsection")],
    strip_headers=False,
)
```
Tradeoff: Preserves document structure; requires knowing document format upfront.

**Semantic chunking (high quality, higher cost)**
Use when: precision matters more than indexing speed; documents contain varied topics
```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

semantic_splitter = SemanticChunker(
    OpenAIEmbeddings(model="text-embedding-3-small"),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=90,
)
```
Tradeoff: Produces semantically coherent chunks; costs embedding calls during indexing.

**Rule of thumb:**
- Corpus < 1000 docs, precision critical → semantic chunking
- Corpus 1000-100k docs, mixed content → recursive/document-aware
- Corpus > 100k docs, uniform content → fixed-size for indexing speed

### Step 4 — Set up the vector store

**Qdrant (recommended for production, self-hosted or cloud)**
```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams
from langchain_qdrant import QdrantVectorStore
from langchain_openai import OpenAIEmbeddings

client = QdrantClient(url="http://localhost:6333")  # or use QdrantClient(":memory:") for testing

client.recreate_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

vector_store = QdrantVectorStore(
    client=client,
    collection_name="documents",
    embedding=OpenAIEmbeddings(model="text-embedding-3-small"),
)
```

**Chroma (recommended for local development and prototypes)**
```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

vector_store = Chroma(
    collection_name="documents",
    embedding_function=OpenAIEmbeddings(model="text-embedding-3-small"),
    persist_directory="./chroma_db",
)
```

**Pinecone (recommended for serverless / managed)**
```python
from pinecone import Pinecone, ServerlessSpec
from langchain_pinecone import PineconeVectorStore

pc = Pinecone(api_key=os.environ["PINECONE_API_KEY"])
pc.create_index(
    name="documents",
    dimension=1536,
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1"),
)
vector_store = PineconeVectorStore(
    index=pc.Index("documents"),
    embedding=OpenAIEmbeddings(model="text-embedding-3-small"),
)
```

### Step 5 — Build the indexing pipeline

Always add metadata to every chunk — it is essential for filtering and citations:

```python
from langchain_core.documents import Document
from pathlib import Path

def load_and_chunk(file_path: str, splitter) -> list[Document]:
    """Load a file, chunk it, and attach source metadata to every chunk."""
    path = Path(file_path)
    raw_text = path.read_text(encoding="utf-8")
    chunks = splitter.split_text(raw_text)
    return [
        Document(
            page_content=chunk,
            metadata={
                "source": str(path),
                "filename": path.name,
                "chunk_index": i,
                "total_chunks": len(chunks),
            },
        )
        for i, chunk in enumerate(chunks)
    ]

def index_documents(file_paths: list[str], vector_store, splitter) -> int:
    """Index all documents and return the count of chunks created."""
    all_docs: list[Document] = []
    for path in file_paths:
        all_docs.extend(load_and_chunk(path, splitter))
    vector_store.add_documents(all_docs)
    return len(all_docs)
```

### Step 6 — Build the retrieval chain

**Basic semantic retrieval:**
```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

retriever = vector_store.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 6},
)

prompt = ChatPromptTemplate.from_template("""
Answer the question using only the provided context. If the answer is not in the
context, say "I don't have enough information to answer this."

Context:
{context}

Question: {question}

Cite the source filename for each claim using [filename] notation.
""")

def format_docs(docs: list[Document]) -> str:
    return "\n\n---\n\n".join(
        f"Source: {d.metadata['filename']}\n{d.page_content}" for d in docs
    )

chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | ChatOpenAI(model="gpt-4o-mini", temperature=0)
    | StrOutputParser()
)

answer = chain.invoke("What is the refund policy?")
```

**Hybrid search (dense + sparse, recommended for production):**
```python
from qdrant_client.models import SparseVector, NamedSparseVector
from langchain_qdrant import QdrantVectorStore, RetrievalMode

# Requires Qdrant with sparse vector support
vector_store = QdrantVectorStore(
    client=client,
    collection_name="documents",
    embedding=OpenAIEmbeddings(model="text-embedding-3-small"),
    retrieval_mode=RetrievalMode.HYBRID,
    sparse_embedding=FastEmbedSparse(model_name="Qdrant/bm25"),
)
```

**Reranking (add after retrieval for precision-critical use cases):**
```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain_cohere import CohereRerank

reranker = CohereRerank(model="rerank-english-v3.0", top_n=3)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=retriever,
)
```

### Step 7 — Project structure

```
rag-pipeline/
├── pyproject.toml
├── src/
│   └── rag/
│       ├── __init__.py
│       ├── indexer.py        # Document loading, chunking, embedding, upsert
│       ├── retriever.py      # Vector store init, retrieval chain construction
│       ├── chain.py          # LangChain chain assembly, prompt template
│       └── config.py         # Pydantic Settings (vector store URL, model names)
├── scripts/
│   └── index.py              # CLI: python -m scripts.index --dir ./docs
└── tests/
    ├── test_indexer.py
    └── test_retriever.py
```

---

## Examples

### Example 1 — Local PDF Q&A with Chroma

**User input:** "I have a folder of PDF reports. I want to ask questions about them
locally without sending data to a cloud service."

**Stack decision:** Chroma (local persist) + Ollama `nomic-embed-text` + local LLM via Ollama

```python
from langchain_community.document_loaders import PyMuPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_ollama import OllamaEmbeddings, ChatOllama

# Indexing
loader = PyMuPDFLoader("reports/q3_financials.pdf")
docs = loader.load()
splitter = RecursiveCharacterTextSplitter(chunk_size=600, chunk_overlap=80)
chunks = splitter.split_documents(docs)

embeddings = OllamaEmbeddings(model="nomic-embed-text")
db = Chroma.from_documents(chunks, embeddings, persist_directory="./local_db")

# Retrieval
retriever = db.as_retriever(search_kwargs={"k": 5})
llm = ChatOllama(model="llama3.1:8b", temperature=0)
```

**Total external API calls:** 0. Fully air-gapped.

---

### Example 2 — Multi-tenant RAG with Metadata Filtering

**User input:** "I have documents from multiple clients. Users should only retrieve
documents from their own account."

**Key addition — metadata filter at query time:**
```python
def get_retriever(user_id: str, vector_store: QdrantVectorStore):
    """Return a retriever scoped to a single user's documents."""
    from qdrant_client.models import Filter, FieldCondition, MatchValue
    return vector_store.as_retriever(
        search_kwargs={
            "k": 6,
            "filter": Filter(
                must=[FieldCondition(key="user_id", match=MatchValue(value=user_id))]
            ),
        }
    )

# Indexing: always attach user_id to chunk metadata
Document(page_content=chunk, metadata={"source": path, "user_id": "acct_123"})
```

---

### Example 3 — Evaluating Retrieval Quality

**After building the pipeline, measure retrieval recall before deploying:**

```python
# Minimal retrieval evaluation without external eval framework
test_cases = [
    {"question": "What is the cancellation policy?", "expected_source": "terms.pdf"},
    {"question": "How do I reset my password?", "expected_source": "faq.pdf"},
]

def evaluate_retrieval(retriever, test_cases: list[dict]) -> dict:
    hits = 0
    for case in test_cases:
        docs = retriever.invoke(case["question"])
        sources = [d.metadata["filename"] for d in docs]
        if case["expected_source"] in sources:
            hits += 1
    return {"recall_at_k": hits / len(test_cases), "k": retriever.search_kwargs["k"]}

print(evaluate_retrieval(retriever, test_cases))
# {"recall_at_k": 0.87, "k": 6}
```

---

## Anti-patterns

1. **Chunking without overlap.** Setting `chunk_overlap=0` means a sentence that spans
   a chunk boundary is split in half. The embedding for each chunk misses half the
   context of that sentence, and retrieval quality for questions about that content
   drops sharply. Always use at least 10-15% overlap relative to chunk size.

2. **Storing full documents in a single chunk.** Embedding a 50-page PDF as one vector
   produces a useless average embedding that retrieves nothing precisely. Chunk to
   256-1024 tokens depending on content density. Verify chunk quality by printing
   5 random chunks before indexing the full corpus.

3. **Not storing source metadata on chunks.** Without `source`, `page`, or `chunk_index`
   metadata, you cannot cite sources in the answer or debug why a bad chunk was retrieved.
   Every chunk must carry enough metadata to identify and re-fetch its original document.

4. **Using k=3 retrieval for all queries.** The right value of k depends on the query
   type and document density. For precise factual queries, k=3 may be fine. For
   comparative questions ("how does policy A differ from policy B?"), k=3 may miss
   one policy entirely. Profile your test queries and tune k per retriever use case,
   or use a reranker and retrieve k=20 then compress to top 3.

5. **Skipping retrieval evaluation before shipping.** A pipeline that builds without
   errors is not a pipeline that retrieves correctly. Always run a minimum 20-case
   recall evaluation (correct source in top-k results) before deploying to users.
   A retrieval recall below 0.80 at k=6 indicates a chunking or embedding model
   problem that more LLM prompt engineering will not fix.
