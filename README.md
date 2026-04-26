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
    * `./chroma_data:/chroma/chroma` in the compose file ensures that even if the container is deleted, your "memory" remains on your physical disk.
* **Accessing the UI:** Open `localhost:7860` to access the LangFlow dashboard.
    * **Opening Browser in VS Code:**
        1. Press `Cmd+Shift+P` (Mac) or `Ctrl+Shift+P` (Windows/Linux)
        2. Type "Simple Browser: Show"
        3. Select it from the dropdown
        4. Paste `http://localhost:7860` in the address bar
        5. Press Enter
    * **Alternative:** Use the Live Preview extension from the marketplace for more features.

---

## III. Live Demo: Building the RAG Pipeline from Scratch (00:25 – 00:55)
**Goal:** Step-by-step construction of the RAG flow starting from a **Blank Flow**.

> **Important:** We are intentionally **NOT** using a template. Students learn best by building each component themselves and understanding *why* each connection exists.

### Setup: Create a Blank Flow
1. From the LangFlow dashboard, click **"Create first flow"** (or **"+ New Flow"**).
2. In the template dialog, scroll down and select **"Blank Flow"**.
3. You will see an empty canvas — this is where we build our RAG pipeline piece by piece.
4. **How to add components:** Drag from the left sidebar onto the canvas. **How to connect:** Click and drag from one node's output handle to another node's input handle.

---

### 1. The Ingestion Flow (The "Writer")
**Goal:** Load a document, split it into chunks, and store it in ChromaDB.

* **Components to add (in order):**
    1. **File Loader** — from the *Data* category. Configure it to accept a PDF or TXT file.
    2. **Recursive Character Text Splitter** — from the *Processing* category. Set `chunk_size` to ~1000 and `chunk_overlap` to ~100.
    3. **OpenAI Embeddings** — from the *Embeddings* category. Use `text-embedding-3-small`.
    4. **Chroma DB** — from the *Vector Stores* category. Set `Collection Name` to `rag_demo` and `Host` to `chromadb` (the Docker service name) on port `8000`.
* **Connections:** `File Loader` → `Text Splitter` → `Chroma DB` (and connect `OpenAI Embeddings` → `Chroma DB`).
* **Speaking Point:** "We split text into chunks because LLMs have a 'context window' limit. We use an overlap (e.g., 10%) so we don't lose context at the edges of a cut."
* **Run it:** Click the play button on the `Chroma DB` node to ingest the document.

---

### 2. The Retrieval Flow (The "Reader")
**Goal:** Take a user question, embed it, and find matching chunks.

* **Components to add (in order):**
    1. **Chat Input** — from the *Inputs* category.
    2. **OpenAI Embeddings** — *the exact same model* as above (`text-embedding-3-small`).
    3. **Chroma DB (Search Mode)** — same `Collection Name` (`rag_demo`) and `Host` as the ingestion flow. Switch the mode to **"Search"** or use a dedicated retriever node.
* **Connections:** `Chat Input` → `OpenAI Embeddings` → `Chroma DB (Search)`.
* **Logic:** The user query is embedded into the same vector space as our documents to find a match.
* **Speaking Point:** "Notice we use the *same* embedding model on both sides. Mismatched models = nonsense results."

---

### 3. The Augmentation & Generation
**Goal:** Combine the retrieved chunks with the question and let the LLM answer.

* **Components to add (in order):**
    1. **Prompt Template** — from the *Prompts* category. Use a template like:
       ```
       Use ONLY the following context to answer the question.
       If the answer is not in the context, say "I don't know."

       Context: {context}
       Question: {question}
       ```
    2. **OpenAI Model** (or any LLM) — from the *Models* category. You can choose:
       * OpenAI: `gpt-4` or `gpt-3.5-turbo`
       * Google Gemini: `gemini-pro`
       * Anthropic Claude: `claude-3-sonnet`
       * Groq: `mixtral-8x7b-32768`
       
       Set `Temperature` to **0.1** for factual responses.
    3. **Chat Output** — from the *Outputs* category.
* **Connections:** `Chroma DB (Search)` results → `Prompt Template` ({context}); `Chat Input` → `Prompt Template` ({question}); `Prompt Template` → `OpenAI Model` → `Chat Output`.
* **Speaking Point:** "The Prompt Template is where the magic happens. We tell the AI: 'Use only the provided context to answer the question.' This is the 'open-book exam' for the LLM."
* **Test it:** Click the **Playground** button (top-right) and ask a question about your uploaded document.

---

## IV. Troubleshooting RAG (00:55 – 01:10)
**Goal:** Address common pitfalls when things don't work as expected.

| Issue | Common Cause | Solution |
| :--- | :--- | :--- |
| **Empty Responses** | Missing or wrong collection name. | Ensure the `Collection Name` in the Ingest and Retrieve nodes match exactly. |
| **Hallucinations** | Model is ignoring context. | Lower the `Temperature` to **0.1** and verify the Prompt Template includes `{context}`. |
| **Docker Connection Refused** | Incorrect Host URL. | Inside Docker, use `http://chromadb:8000` instead of `localhost`. |
| **Embedding Mismatch** | Using different models for Ingest vs. Query. | Always use the **exact same** embedding model (e.g., `text-embedding-3-small`) for both sides. |
| **Data "Disappearing"** | No volume mapping in Docker. | Map a local folder to `/chroma/chroma` in the Chroma container. |

---

## V. Conclusion & Q&A (01:10 – 01:20/80)
* **Summary:** We’ve built a system that doesn't just "talk"—it "researches."
* **Challenge:** Ask the participants to swap the File Loader for a Website Crawler node to see how RAG works with live web data.
* **Open Floor:** Take questions on scaling, security, and metadata filtering.

---

### Instructor Setup Checklist
1.  **Docker Desktop:** Running and healthy.
2.  **VSCode:** Terminal open to the project root.
3.  **API Keys:** Set your LLM provider API key(s) using **one** of these methods:
    * **Method A — Export environment variables (temporary):**
      ```bash
      export OPENAI_API_KEY=your_key_here
      # or
      export GOOGLE_API_KEY=your_key_here
      # or
      export ANTHROPIC_API_KEY=your_key_here
      # or
      export GROQ_API_KEY=your_key_here
      ```
    * **Method B — Create a `.env` file (persistent, safer):**
      Create a file named `.env` in your project root:
      ```
      OPENAI_API_KEY=your_key_here
      # GOOGLE_API_KEY=your_key_here
      # ANTHROPIC_API_KEY=your_key_here
      # GROQ_API_KEY=your_key_here
      ```
      Then start the stack: `docker compose up -d`
4.  **Sample Data:** A 2-page PDF or TXT file ready for the demo.

Does this structured approach align with the technical depth you want to provide for your live session?