
# 🔹 1. Dense vs Sparse Vectors

| Feature           | Dense Vectors                                   | Sparse Vectors                                     |
| ----------------- | ----------------------------------------------- | -------------------------------------------------- |
| Definition        | Most values are **non-zero**                    | Most values are **zero**                           |
| Example           | `[0.12, -0.98, 0.33, 0.44]`                     | `[0, 0, 5, 0, 0, 12, 0]`                           |
| Storage           | Compact (fixed-length arrays)                   | Efficient storage using indices of non-zero values |
| Source            | Neural network embeddings (e.g., transformers)  | Traditional methods (TF-IDF, BM25)                 |
| Semantic Meaning  | Captures **semantic similarity**                | Captures **exact keyword matching**                |
| Typical Dimension | 128–4096+                                       | Can be very large (10k–100k+)                      |
| Speed             | Fast for similarity search                      | Can be slower depending on implementation          |
| Use Case          | Semantic search, recommendations, LLM retrieval | Keyword search, filtering, hybrid search           |

### 👉 When to use

| Scenario                                           | Best Choice             |
| -------------------------------------------------- | ----------------------- |
| Natural language meaning (e.g., “car” ≈ “vehicle”) | Dense                   |
| Exact keyword match (e.g., legal or log search)    | Sparse                  |
| Best of both worlds                                | Hybrid (Dense + Sparse) |

---

# 🔹 2. Vector Dimension

| Concept          | Explanation                                                           |
| ---------------- | --------------------------------------------------------------------- |
| Dimension        | Number of values in a vector                                          |
| Example          | A 768-dim embedding → 768 numbers                                     |
| Source           | Determined by embedding model (e.g., BERT = 768, OpenAI = 1536, etc.) |
| Higher Dimension | More expressive but heavier computation                               |
| Lower Dimension  | Faster but less expressive                                            |

### 👉 Trade-offs

| Dimension Size          | Pros             | Cons                 |
| ----------------------- | ---------------- | -------------------- |
| Low (e.g., 128–384)     | Fast, low memory | Less semantic detail |
| Medium (e.g., 512–1024) | Balanced         | Moderate cost        |
| High (e.g., 1536–4096+) | Rich semantics   | Slower, more storage |

### 👉 When to choose

| Scenario                          | Recommendation                 |
| --------------------------------- | ------------------------------ |
| Real-time systems                 | Lower dimensions               |
| High accuracy semantic search     | Higher dimensions              |
| Mobile / edge deployment          | Smaller vectors                |
| Large-scale retrieval (millions+) | Medium + indexing optimization |

---

# 🔹 3. Distance / Similarity Metrics

| Metric             | Formula Idea                 | What It Measures      | Range     | Key Property           |
| ------------------ | ---------------------------- | --------------------- | --------- | ---------------------- |
| Cosine Similarity  | Angle between vectors        | Direction similarity  | -1 to 1   | Ignores magnitude      |
| Dot Product        | Sum of element-wise products | Direction + magnitude | Unbounded | Sensitive to scale     |
| Euclidean Distance | Straight-line distance       | Absolute distance     | 0 to ∞    | Sensitive to magnitude |

---

## 🔍 Metric Comparison

| Metric      | Best For                                          | Weakness                       |
| ----------- | ------------------------------------------------- | ------------------------------ |
| Cosine      | Text embeddings, semantic similarity              | Ignores vector magnitude       |
| Dot Product | When magnitude matters (e.g., ranking confidence) | Can bias large vectors         |
| Euclidean   | Physical space, clustering                        | Poor for high-dimensional text |

---

# 🔹 4. When to Use Each Metric

| Use Case                               | Recommended Metric            | Why                             |
| -------------------------------------- | ----------------------------- | ------------------------------- |
| Semantic search (text, LLM embeddings) | Cosine                        | Focus on meaning, not magnitude |
| OpenAI / transformer embeddings        | Cosine (or dot if normalized) | Industry standard               |
| Recommendation systems                 | Dot Product                   | Magnitude encodes importance    |
| Clustering                             | Euclidean or Cosine           | Depends on data normalization   |
| Hybrid ranking (BM25 + embeddings)     | Cosine + Sparse scoring       | Combine semantic + keyword      |

---

# 🔹 5. Practical Summary (Cheat Sheet)

| Component       | Default Choice                 | When to Change                                         |
| --------------- | ------------------------------ | ------------------------------------------------------ |
| Vector Type     | Dense                          | Use Sparse for keyword-heavy tasks                     |
| Dimension       | Model default (e.g., 768–1536) | Reduce for speed, increase for quality                 |
| Metric          | Cosine                         | Use dot if vectors are normalized or magnitude matters |
| Search Strategy | Dense similarity               | Use hybrid for best performance                        |

---

# 🔹 6. Real-World Example

| Application                   | Setup                   |
| ----------------------------- | ----------------------- |
| Chatbot retrieval (RAG)       | Dense + Cosine          |
| E-commerce search             | Hybrid (Dense + Sparse) |
| Log search                    | Sparse (BM25)           |
| Music / video recommendations | Dense + Dot Product     |

---
