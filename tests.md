# Experiment Report: Retrieval & Generation Performance

## 1. Baseline Generation Performance
**Configuration:**
- **Retrieval:** TF-IDF (Top 5, Recall: 85.37%)
- **Generator:** gemini-2.5-flash-lite

| Metric | Text Only | Text + Image |
| :--- | :--- | :--- |
| **Average Value Accuracy** | **57.89%** | 42.11% |
| **Average Ref ID Jaccard** | 34.21% | **76.32%** |
| **Average Weighted Score** | **0.5803** | 0.5197 |

> **Note regarding Multimodal (Text + Image):**
> 許多單位上的小問題，與只有text輸出內容相異不大 (flash-lite 太笨、prompt 不夠好)，整體而言餵 image 有助於 generator 讀圖表。

---

## 2. Retrieval Strategy Analysis

### A. Sparse Retrieval (Initial Baseline)
*Baseline Method:* Default TF-IDF / BM25

**Base Results (Top 5 Hits):**
- **TF-IDF:** 85.37%
- **BM25:** 85.37%

**Re-ranking Results:**
*Strategy: Retrieve Top 50 (BM25) → Rerank → Select Top 5*

| Reranker Model | Final Recall | Improvement |
| :--- | :--- | :--- |
| ms-marco-MiniLM-L-6-v2 | 85.37% | - |
| jinaai/jina-reranker-v1-turbo-en | 85.37% | - |
| **BAAI/bge-reranker-v2-m3** | **95.12%** | **+9.75%** |

### B. Dense Retrieval
*Baseline Method:* Embedding Similarity

**Base Results (Top 5 Hits):**
- **all-MiniLM-L6-v2:** 70.73%
- **multi-qa-mpnet-base-dot-v1:** 65.85%
- **nomic-ai/nomic-embed-text-v1.5** (w/o Instruction): 68.29%
- **nomic-ai/nomic-embed-text-v1.5** (w/ Instruction): 78.05%

**Re-ranking Results:**
*Strategy: Retrieve Top 50 (Nomic) → Rerank → Select Top 5*

| Reranker Model | Final Recall | Improvement |
| :--- | :--- | :--- |
| ms-marco-MiniLM-L-6-v2 | 80.49% | +2.44% |
| jinaai/jina-reranker-v1-turbo-en | 87.80% | +9.75% |
| **BAAI/bge-reranker-v2-m3** | **95.12%** | **+17.07%** |

---

## 3. Optimized Sparse Retrieval (Grid Search Results)

針對 Sparse Retrieval 進行參數與分詞優化後的結果。

### Key Improvements
1.  **Tokenizer:** 使用 Regex `\w+` 取代 `.split()`，移除標點符號干擾。
2.  **N-grams:** TF-IDF 開啟 Bigram `(1, 2)` 顯著提升專有名詞識別。
3.  **Parameters:** 調整 BM25 ($k_1, b$) 與 TF-IDF (sublinear_tf, min_df)。

### Grid Search Metrics

| Model | Parameters | Recall@5 | Recall@50 | Note |
| :--- | :--- | :---: | :---: | :--- |
| **TF-IDF (Optimized)** | **ngram=(1,2), min_df=2, sublinear_tf=True** | **1.0000** | **1.0000** | **Perfect Recall** |
| BM25 (Optimized) | $k_1=1.4, b=0.75$ | 0.9487 | 1.0000 | |
| BM25 (Optimized) | $k_1=1.2, b=0.50$ | 0.9231 | 1.0000 | |

> **Observation:**
> *   **Recall@50 達到 100%**：經過優化後，TF-IDF 與 BM25 均能在前 50 篇文檔中召回所有正確答案，這為後續的 Reranker 提供了完美的候選集。
> *   **TF-IDF > BM25**：在本資料集上，TF-IDF 結合 Bigram 的效果超越 BM25，直接在 Top 5 達到 100% 召回率。

---

## 4. Detailed Metrics (Baseline vs Optimized)

### Recall Metrics Comparison

| Model | Recall@1 | Recall@5 | Recall@10 | Recall@20 | Recall@50 |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **TF-IDF (Optimized)** | **0.897** | **1.000** | **1.000** | **1.000** | **1.000** |
| **BM25 (Optimized)** | 0.795 | 0.949 | 0.949 | **1.000** | **1.000** |
| BM25 (Baseline) | 0.744 | 0.897 | 0.923 | 0.949 | 1.000 |
| TF-IDF (Baseline) | 0.744 | 0.897 | 0.923 | 0.974 | 0.974 |
| BGE-M3-Dense | 0.590 | 0.821 | 0.897 | 0.974 | 1.000 |
| Nomic-v1.5 | 0.590 | 0.795 | 0.923 | 0.949 | 1.000 |

### Retriever-Reranker Pipeline Performance
*Base Retriever: BM25 (Baseline)*

| Reranker Type | Reranker Name | Final Recall@1 | Final Recall@5 |
| :--- | :--- | :---: | :---: |
| **Cross Encoder** | **BAAI/bge-reranker-v2-m3** | **0.897** | **0.949** |
| Cross Encoder | cross-encoder/ms-marco-MiniLM-L-6-v2 | 0.667 | 0.872 |
| Cross Encoder | jinaai/jina-reranker-v1-turbo-en | 0.615 | 0.846 |
| Bi-Encoder | jinaai/jina-embeddings-v2-base-en | 0.641 | 0.795 |
| Bi-Encoder | Alibaba-NLP/gte-large-en-v1.5 | 0.615 | 0.718 |
