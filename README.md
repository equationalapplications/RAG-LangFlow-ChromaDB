# Lesson Plan: RAG using LangFlow and ChromaDB
**Total Duration:** 60 – 80 Minutes  
**Focus:** Building a local, persistent Retrieval-Augmented Generation pipeline using containerized infrastructure.

---

## I. Introduction: The Power of RAG (00:00 – 00:10)
**Goal:** Define RAG and its necessity in modern AI applications.

* **What is RAG?** Explain that RAG (Retrieval-Augmented Generation) is the process of providing an LLM with specific, external data to answer questions, reducing hallucinations and overcoming "knowledge cutoffs."
* **The Workflow:** 1.  **Ingest:** Documents are broken into chunks.
    2.  **Embed:** Chunks are converted into vectors ($v \in \mathbb{R}^n$).
    3.  **Retrieve:** At query time, the system finds the most similar vectors in **ChromaDB**.
    4.  **Augment:** These snippets are added to the LLM prompt.
    5.  **Generate:** The LLM produces a fact-based response.



---

## II. Environment Setup (00:10 – 00:25)
**Goal:** Establish the local development stack.

* **VSCode & Docker Compose:**
    * Show a `docker-compose.yml` that links LangFlow and ChromaDB.
    * **Start the Stack:** Run `docker compose up -d` to start both services in detached mode.
    * **Technical Step:** Highlight the network configuration. LangFlow needs to talk to ChromaDB via the service name (e.g., `http://chromadb:8000`).
* **Persistence:** Demonstrate the use of Docker Volumes.
    * `./chroma_data:/data` in the compose file ensures that even if the container is deleted, your "memory" remains on your physical disk.
* **Accessing the UI:** Open `localhost:7860` to access the LangFlow dashboard.

---

## III. Live Demo: Building the RAG Pipeline (00:25 – 00:55)
**Goal:** Step-by-step construction of the RAG flow.

### 1. The Ingestion Flow (The "Writer")
* **Components:** `File Loader` $\rightarrow$ `Recursive Character Text Splitter` $\rightarrow$ `ChromaDB`.
* **Speaking Point:** "We split text into chunks because LLMs have a 'context window' limit. We use an overlap (e.g., 10%) so we don't lose context at the edges of a cut."

### 2. The Retrieval Flow (The "Reader")
* **Components:** `Chat Input` $\rightarrow$ `OpenAI Embeddings` $\rightarrow$ `ChromaDB (Search)`.
* **Logic:** The user query is embedded into the same vector space as our documents to find a match.

### 3. The Augmentation & Generation
* **Components:** `Prompt Template` $\rightarrow$ `OpenAI Model` $\rightarrow$ `Chat Output`.
* **Speaking Point:** "The Prompt Template is where the magic happens. We tell the AI: 'Use only the provided context to answer the question.' This is the 'open-book exam' for the LLM."

---

## IV. Troubleshooting RAG (00:55 – 01:10)
**Goal:** Address common pitfalls when things don't work as expected.

| Issue | Common Cause | Solution |
| :--- | :--- | :--- |
| **Empty Responses** | Missing or wrong collection name. | Ensure the `Collection Name` in the Ingest and Retrieve nodes match exactly. |
| **Hallucinations** | Model is ignoring context. | Lower the `Temperature` to **0.1** and verify the Prompt Template includes `{context}`. |
| **Docker Connection Refused** | Incorrect Host URL. | Inside Docker, use `http://chromadb:8000` instead of `localhost`. |
| **Embedding Mismatch** | Using different models for Ingest vs. Query. | Always use the **exact same** embedding model (e.g., `text-embedding-3-small`) for both sides. |
| **Data "Disappearing"** | No volume mapping in Docker. | Map a local folder to `/index_data` in the Chroma container. |

---

## V. Conclusion & Q&A (01:10 – 01:20/80)
* **Summary:** We’ve built a system that doesn't just "talk"—it "researches."
* **Challenge:** Ask the participants to swap the File Loader for a Website Crawler node to see how RAG works with live web data.
* **Open Floor:** Take questions on scaling, security, and metadata filtering.

---

### Instructor Setup Checklist
1.  **Docker Desktop:** Running and healthy.
2.  **VSCode:** Terminal open to the project root.
3.  **API Keys:** OpenAI or Ollama environment variables set.
4.  **Sample Data:** A 2-page PDF or TXT file ready for the demo.

Does this structured approach align with the technical depth you want to provide for your live session?