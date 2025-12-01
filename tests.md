# Sparse Retrieval

### Base Results
- TF-IDF: 85.37%
- BM25: 85.37%

### Re-ranking Strategy
> Take top 50 hits on Train_QA, then re-rank and select top 5 hits (BM25)

- ms-marco-MiniLM-L-6-v2: 85.37%
- jinaai/jina-reranker-v1-turbo-en: 85.37%
- BAAI/bge-reranker-v2-m3: 95.12%

---

# Dense Retrieval

### Base Results
Take top 5 hits on Train_QA

- all-MiniLM-L6-v2: 70.73%
- multi-qa-mpnet-base-dot-v1: 65.85%
- nomic-ai/nomic-embed-text-v1.5 (without Instruction Tuning): 68.29%
- nomic-ai/nomic-embed-text-v1.5 (with Instruction Tuning): 78.05%

### Re-ranking Strategy
> Take top 50 hits on Train_QA, then re-rank and select top 5 hits (nomic)

- ms-marco-MiniLM-L-6-v2: 80.49%
- jinaai/jina-reranker-v1-turbo-en: 87.8%
- BAAI/bge-reranker-v2-m3: 95.12%

---

# Recall Metrics by Model

| Model | Recall@1 | Recall@5 | Recall@10 | Recall@20 | Recall@50 |
|-------|----------|----------|-----------|-----------|-----------|
| TF-IDF | 0.744 | 0.897 | 0.923 | 0.974 | 0.974 |
| BM25 | 0.744 | 0.897 | 0.923 | 0.949 | 1.000 |
| BGE-M3-Sparse | 0.667 | 0.795 | 0.897 | 0.949 | 1.000 |
| BGE-M3-Dense | 0.590 | 0.821 | 0.897 | 0.974 | 1.000 |
| Nomic-v1.5 | 0.590 | 0.795 | 0.923 | 0.949 | 1.000 |
| MiniLM-L6 | 0.513 | 0.718 | 0.769 | 0.872 | 0.923 |

---

# Retriever-Reranker Combinations

| Retriever | Reranker Type | Reranker Name | Final Recall@1 | Final Recall@5 |
|-----------|---------------|---------------|----------------|----------------|
| BM25 | Cross | cross-encoder/ms-marco-MiniLM-L-6-v2 | 0.667 | 0.872 |
| BM25 | Cross | BAAI/bge-reranker-v2-m3 | 0.897 | 0.949 |
| BM25 | Cross | jinaai/jina-reranker-v1-turbo-en | 0.615 | 0.846 |
| BM25 | Bi | jinaai/jina-embeddings-v2-base-en | 0.641 | 0.795 |
| BM25 | Bi | Alibaba-NLP/gte-large-en-v1.5 | 0.615 | 0.718 |
