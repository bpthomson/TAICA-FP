# Experiment Report: Retrieval & Generation Performance

## 1. Baseline Generation Performance

**Configuration**
- **Retrieval:** TF-IDF (Top 5, Recall@5: 87%)
- **Generator:** gemini-2.5-flash-lite

| Metric                    | Text Only | Text + Image |
| :------------------------ | :-------: | :----------: |
| **Average Value Accuracy**    | **58%**   | 42%          |
| **Average Ref ID Jaccard**   | 34%       | **76%**      |
| **Average Weighted Score**   | **58%**   | 52%          |

> Text+Image其實沒比較差，許多數值與 text-only 輸出相近，都是單位上的小問題，與只有text輸出內容相異不大 (flash-lite 太笨、prompt 不夠好)  
> 整體而言餵 image 有助於 generator 讀圖表。

---

## 2. Retrieval Strategy Analysis

### A. Sparse Retrieval

**Base Results (Top 5 hits)**  
- **TF-IDF:** 87%  
- **BM25:** 89%

**Re-ranking Results**  
*Strategy: Retrieve Top 50 (BM25) → Rerank → Select Top 5*

| Reranker Model                     | Final Recall@5 | Improvement |
| :--------------------------------- | :------------: | :---------: |
| ms-marco-MiniLM-L-6-v2            | 90%            | –           |
| jinaai/jina-reranker-v1-turbo-en  | 90%            | –           |
| **BAAI/bge-reranker-v2-m3**       | **95%**        | **+5%**     |

### B. Dense Retrieval

**Base Results (Top 5 hits)**  
- **all-MiniLM-L6-v2:** 71%  
- **multi-qa-mpnet-base-dot-v1:** 66%  
- **nomic-ai/nomic-embed-text-v1.5 (w/o Instruction):** 68%  
- **nomic-ai/nomic-embed-text-v1.5 (w/ Instruction):** 80%
- **BGE-M3-Dense:** 82%

**Re-ranking Results**  
*Strategy: Retrieve Top 50 (Nomic) → Rerank → Select Top 5*

| Reranker Model                     | Final Recall@5 | Improvement |
| :--------------------------------- | :------------: | :---------: |
| ms-marco-MiniLM-L-6-v2            | 80%            | -         |
| jinaai/jina-reranker-v1-turbo-en  | 88%            | -        |
| **BAAI/bge-reranker-v2-m3**       | **95%**        | **+17%**    |

---

### Retriever–Reranker Pipeline Performance

*Base retriever: BM25*

| Reranker Type | Reranker Name                          | Final Recall@1 | Final Recall@5 |
| :------------ | :-------------------------------------- | :------------: | :------------: |
| **Cross Encoder** | **BAAI/bge-reranker-v2-m3**         | **90%**        | **95%**        |
| Cross Encoder | cross-encoder/ms-marco-MiniLM-L-6-v2   | 67%            | 87%            |
| Cross Encoder | jinaai/jina-reranker-v1-turbo-en       | 62%            | 85%            |
| Bi-Encoder    | jinaai/jina-embeddings-v2-base-en      | 64%            | 80%            |
| Bi-Encoder    | Alibaba-NLP/gte-large-en-v1.5          | 62%            | 72%            |

---

## 3. Optimized Sparse Retrieval (Grid Search Results)

針對 Sparse Retrieval（TF-IDF、BM25）進行分詞及參數的 Grid Search，觀察不同設定下的 Recall@K 表現。

### 3.1 TF-IDF Grid Search (Full Metrics)

| ngram_range | sublinear_tf | max_df | min_df | Recall@1 | Recall@5 | Recall@10 | Recall@20 | Recall@50 |
| :---------: | :----------: | :----: | :----: | :------: | :------: | :-------: | :-------: | :-------: |
| (1, 1)      | False        | 0.95   | 2      | 74%      | 92%      | 92%       | **97%**   | **100%**  |
| (1, 1)      | False        | 0.95   | 1      | 72%      | 87%      | **95%**   | 95%       | **100%**  |
| (1, 2)      | True         | 1.00   | 2      | **87%**  | **95%**  | **95%**   | **97%**   | 97%       |
| (1, 2)      | True         | 0.95   | 2      | **87%**  | **95%**  | **95%**   | **97%**   | 97%       |
| (1, 2)      | False        | 1.00   | 1      | 82%      | **95%**  | **95%**   | **97%**   | 97%       |
| (1, 2)      | False        | 1.00   | 2      | **87%**  | **95%**  | **95%**   | **97%**   | 97%       |
| (1, 2)      | False        | 0.95   | 2      | 85%      | **95%**  | **95%**   | **97%**   | 97%       |
| (1, 1)      | True         | 1.00   | 2      | 67%      | 92%      | **95%**   | 95%       | 97%       |
| (1, 1)      | True         | 0.95   | 2      | 67%      | 92%      | **95%**   | 95%       | 97%       |
| (1, 1)      | False        | 1.00   | 2      | 74%      | 92%      | **95%**   | **97%**   | 97%       |

- bigram在低 top‑k（例如 Recall@1）表現較佳。
- 不過 unigram 且 sublinear_tf=False的設定可以在 Recall@50 達到 **100%**，最終選用此組合作為優化後 TF-IDF。

### 3.2 BM25 Grid Search

| k1   | b    | Recall@1 | Recall@5 | Recall@10 | Recall@20 | Recall@50 |
| :--: | :--: | :------: | :------: | :-------: | :-------: | :-------: |
| 1.2  | 0.75 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.2  | 0.80 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.2  | 0.90 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.4  | 0.75 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.4  | 0.80 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.4  | 0.90 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.5  | 0.60 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.5  | 0.75 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.5  | 0.80 | **77%**  | **92%**  | **92%**   | **95%**   | **97%**   |
| 1.5  | 0.90 | 74%      | **92%**  | **92%**   | **95%**   | **97%**   |

- 不同 BM25 參數組合之間差異極小。
- 即便在最佳設定下，BM25 表現仍略遜於優化後的 TF-IDF，因此後續研究重心轉向 TF-IDF。

---

## 4. Detailed Metrics (Baseline vs Optimized)

### 4.1 Recall Metrics Comparison

| Model              | Recall@1 | Recall@5 | Recall@10 | Recall@20 | Recall@50 |
| :----------------- | :------: | :------: | :-------: | :-------: | :-------: |
| **TF-IDF (Optimized)** | 74%      | **92%**  | **92%**   | **97%**   | **100%**  |
| **BM25 (Optimized)**   | **77%**  | **92%**  | **92%**   | 95%       | 97%       |
| BM25 (Baseline)    | 74%      | 90%      | **92%**   | 95%       | 97%       |
| TF-IDF (Baseline)  | 72%      | 87%      | **92%**   | 95%       | **100%**  |
| BGE-M3-Dense       | 59%      | 82%      | 90%       | **97%**   | **100%**  |
| Nomic-v1.5         | 59%      | 80%      | **92%**   | 95%       | **100%**  |
