---
title: "RAG Improvements"
date: 2025-10-25T16:29:51-04:00
draft: false
---



## A Basic Pipeline
A Retrieval-Augmented Generation (RAG) pipeline consists of four main parts: ingestion, storage, retrieval, and generation. To optimize it, start with a simple local setup using LangChain 1.0 and lightweight models. This baseline will serve as the foundation for all further experiments.

The goal is to build a fully local RAG pipeline with a local embedding model, a vector database (FAISS or Chroma), a local LLM such as Ollama, and a small text dataset. This setup lets you measure latency, memory use, and context quality, debug each component independently, and verify end-to-end ingestion and retrieval.

The pipeline works as follows. First, load documents from local files using a LangChain document loader. Next, split the text into fixed-size chunks with no overlap. Convert each chunk into a dense vector using the embedding model. Store these vectors in a vector database. For queries, retrieve the top-K most similar chunks and feed them to a local LLM to generate answers.

```Python
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langchain_community.embeddings import SentenceTransformerEmbeddings
from langchain_community.llms import Ollama
from langchain.chains import RetrievalQA

# Load documents
loader = TextLoader("docs/sample.txt")
documents = loader.load()

# Split into chunks
splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=0)
chunks = splitter.split_documents(documents)

# Create embeddings
embedding_model = SentenceTransformerEmbeddings(model_name="all-MiniLM-L6-v2")

# Build vector store
vectorstore = FAISS.from_documents(chunks, embedding_model)

# Create retriever
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# Local LLM
llm = Ollama(model="mistral")

# Construct retrieval-augmented chain
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=retriever,
)

# Run a query
query = "Summarize the main argument in the document."
response = qa_chain.invoke(query)
print(response["result"])
```

This baseline covers the essential RAG workflow. Ingestion is straightforward, using text only with no metadata. Retrieval relies on vector-based cosine similarity with default settings. Generation uses simple context stuffing without prompt optimization. You can benchmark this setup for end-to-end latency, recall quality, and embedding throughput. These metrics establish a reference point for improvements such as advanced chunking strategies, metadata usage, and refined retrieval logic. The following section will explore recursive, semantic, and sliding-window chunking to enhance recall and reduce context fragmentation.


## Hierarchical Retrieval
Flat retrieval systems do not scale well. As the dataset grows, similarity search becomes slow and returns irrelevant results. More hardware does not fix the problem; better structure does. Hierarchical retrieval adds stages to the search process. The first stage finds broad areas of relevance. The second stage searches within those areas for finer details.

The first stage retrieves large chunks such as full documents or long sections. These provide coverage of general topics. The second stage searches within those top results using smaller chunks, such as paragraphs or sentences, to find precise matches. This approach reduces the number of comparisons and improves accuracy by discarding irrelevant material early.

The method balances precision and recall. Large chunks capture more context but are less specific. Small chunks are more specific but can lose context. Two-stage retrieval combines both. It is also faster: instead of searching every sub-chunk in a large corpus, the first stage narrows the candidates to a small set. This is useful for technical or multi-domain data where high-level grouping matters.

The following example shows a two-level retriever built with LangChain 1.0 and FAISS. The first retriever indexes full documents. The second indexes smaller parts within the top results.

```Python
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import SentenceTransformerEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_community.llms import Ollama

# Stage 1: Coarse retrieval
coarse_loader = TextLoader("docs/knowledge_base.txt")
coarse_docs = coarse_loader.load()

coarse_splitter = RecursiveCharacterTextSplitter(chunk_size=4000, chunk_overlap=0)
coarse_chunks = coarse_splitter.split_documents(coarse_docs)

embedding_model = SentenceTransformerEmbeddings(model_name="all-MiniLM-L6-v2")

coarse_store = FAISS.from_documents(coarse_chunks, embedding_model)
coarse_retriever = coarse_store.as_retriever(search_kwargs={"k": 5})

# Stage 2: Fine retrieval
fine_splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)

def hierarchical_retrieve(query: str):
    coarse_results = coarse_retriever.get_relevant_documents(query)
    fine_docs = []
    for doc in coarse_results:
        fine_docs.extend(fine_splitter.split_text(doc.page_content))
    fine_store = FAISS.from_texts(fine_docs, embedding_model)
    fine_retriever = fine_store.as_retriever(search_kwargs={"k": 4})
    return fine_retriever.get_relevant_documents(query)

# Stage 3: Response generation
llm = Ollama(model="mistral")

def answer_query(query):
    fine_chunks = hierarchical_retrieve(query)
    context = "\n\n".join([chunk.page_content for chunk in fine_chunks])
    prompt = f"Use the following context to answer:\n\n{context}\n\nQuestion: {query}"
    return llm.invoke(prompt)

response = answer_query("How does the cache invalidation logic work?")
print(response)
```

For better ranking, use a CrossEncoder to score the top results. It evaluates query and document pairs together for higher accuracy, at the cost of more computation.

```Python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank_chunks(query, docs, top_k=3):
    pairs = [(query, d.page_content) for d in docs]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(docs, scores), key=lambda x: x[1], reverse=True)
    return [doc for doc, _ in ranked[:top_k]]
```

This combination—FAISS for filtering and a CrossEncoder for re-ranking—offers high precision while keeping runtime low.

Compared to flat retrieval, hierarchical retrieval is faster and more accurate. Latency drops sharply for large datasets. Top-answer relevance improves by roughly 10–20%. Storage costs decrease because only a small subset of text is embedded at fine granularity. For large document collections, this approach gives better performance and higher-quality answers than any single-stage retriever.
