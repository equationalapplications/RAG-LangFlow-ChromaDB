# RAG Chatbot: Step-by-Step Lesson
### LangFlow + ChromaDB + OpenAI — Build from Blank, End with a Working Chatbot

---

## What You'll Build

A chatbot that reads a document you upload. Ask it a question → it searches the document → answers from the content.

```
                  ┌─────────────────────────────────────────────────────┐
  INGEST          │  File → Split Text → Chroma DB ← OpenAI Embeddings  │
                  └─────────────────────────────────────────────────────┘
                                          ↕ same component
                  ┌─────────────────────────────────────────────────────┐
  RETRIEVE+CHAT   │  Chat Input → Chroma DB → Parse Data → Prompt       │
                  │                                     ↗               │
                  │               Chat Input ──────────                 │
                  │                                     ↘               │
                  │                              OpenAI → Chat Output    │
                  └─────────────────────────────────────────────────────┘
```

The **Chroma DB** node does double duty: it stores your document chunks AND searches them when you ask a question.

---

## Prerequisites

- **Docker Desktop** installed and running (docker.com/products/docker-desktop)
- An OpenAI API key (starts with `sk-`, from platform.openai.com)
- This repo cloned to your machine

---

## Step 1 — Start the Containers (2 min)

In your terminal, from the project folder:

```bash
docker compose up -d
```

This starts two containers:
- **LangFlow** — the visual flow builder (port 7860)
- **ChromaDB** — the vector database (port 8000)

Wait about 30 seconds for both to finish starting, then open your browser to **http://localhost:7860**

You should see the LangFlow dashboard.

> To stop everything when you're done: `docker compose down`
> Your flows and data are saved in the `langflow_data/` and `chroma_data/` folders — they survive restarts.

---

## Step 2 — Prepare a Test Document (2 min)

Create a plain text file called `test_doc.txt` with unique, easy-to-verify content:

```
The Meridian Protocol is a set of internal guidelines created in 2019.
It requires all project managers to submit weekly status reports by Friday 5pm.
The protocol was written by Sandra Okafor, Director of Operations.
Violations of the protocol result in a formal review process.
```

Using unique, invented facts makes it easy to confirm the chatbot is actually reading your document and not making things up.

---

## Step 3 — Create a Blank Flow

1. In LangFlow, click **"+ New Flow"** (or **"Create first flow"**)
2. In the dialog that appears, scroll to the bottom and click **"Blank Flow"**
3. You see an empty canvas — this is where you'll build

The **left sidebar** is your component library. There's a search bar at the top. You'll drag components from the sidebar onto the canvas.

---

## Step 4 — Add 9 Components to the Canvas

Search for each component by name in the sidebar search bar, then drag it onto the canvas. Arrange them loosely — you'll reposition after connecting.

Suggested layout:

```
[File]      [Split Text]    [OpenAI Embeddings]    [Chroma DB]
                                                        ↓
[Chat Input]                                       [Parse Data]
                                                        ↓
                                                    [Prompt]
                                                        ↓
                                                    [OpenAI]
                                                        ↓
                                                 [Chat Output]
```

| # | Search for | Category in sidebar |
|---|-----------|---------------------|
| 1 | `File` | **Data** |
| 2 | `Split Text` | **Processing** |
| 3 | `OpenAI Embeddings` | **Embeddings** |
| 4 | `Chroma DB` | **Vector Stores** |
| 5 | `Parse Data` | **Helpers** or **Processing** |
| 6 | `Chat Input` | **Inputs** |
| 7 | `Prompt` | **Prompts** |
| 8 | `OpenAI` | **Models** |
| 9 | `Chat Output` | **Outputs** |

> If you can't find `OpenAI Embeddings` by that name, search for `Embeddings` and look for the option with an OpenAI logo.

---

## Step 5 — Configure Each Component

Click each component to open its settings panel on the right.

### File
- Click the upload area and upload your `test_doc.txt`
- Everything else: leave as default

### Split Text
- **Chunk Size**: `1000`
- **Chunk Overlap**: `200`
- Everything else: leave as default

> "Chunk size" is how many characters per piece. "Overlap" means each chunk shares 200 characters with the next — so context isn't lost at a cut boundary.

### OpenAI Embeddings
- **API Key**: paste your OpenAI API key (`sk-...`)
- **Model**: `text-embedding-3-small` (should already be selected)

> Embeddings convert text into numbers. The same model must be used for both storing AND searching — mixing models produces garbage results.

### Chroma DB
- **Collection Name**: `my_docs`
- **Chroma Server Host**: `chromadb`
- **Chroma Server HTTP Port**: `8000`
- **Number of Results**: `4`

> `chromadb` is the name of the ChromaDB container. LangFlow and ChromaDB are on the same Docker network, so this hostname resolves automatically.

### Parse Data
- **Template**: `{text}`

> Converts the raw search results (structured data objects) into plain text the Prompt can use.

### Chat Input
- No configuration needed

### Prompt
- Click the **"Edit"** or **"Template"** button inside the component
- Replace whatever is there with this exact text:

```
You are a helpful assistant. Answer the question using only the context below.
If the answer is not in the context, say "I don't know based on the document."

Context:
{context}

Question:
{question}
```

- Click **Save** or **Apply**

> After saving, two input ports appear on the Prompt component: `context` and `question`. These are where you connect the other components.

### OpenAI (Chat Model)
- **API Key**: paste your OpenAI API key (same one as above)
- **Model Name**: `gpt-4o-mini`
- **Temperature**: `0.1`

> `gpt-4o-mini` is fast and inexpensive. Temperature `0.1` means "stick to facts, don't be creative."

### Chat Output
- No configuration needed

---

## Step 6 — Wire Everything Together

Connect components by clicking an **output dot** (right side of a component) and dragging to an **input dot** (left side of the next component). Dots are colored — matching colors connect; mismatched types won't snap.

Make these **9 connections** in order:

**Ingestion chain** (loads your document into ChromaDB):

1. **File** → `Data` output → **Split Text** → `Data Input`
2. **Split Text** → `Data` output → **Chroma DB** → `Ingest Data`
3. **OpenAI Embeddings** → `Embeddings` output → **Chroma DB** → `Embedding`

**Retrieval chain** (searches ChromaDB with the user's question):

4. **Chat Input** → `Message` output → **Chroma DB** → `Search Query`
5. **Chroma DB** → `Search Results` output → **Parse Data** → `Data`
6. **Parse Data** → `Text` output → **Prompt** → `context`

**Answer chain** (generates the response):

7. **Chat Input** → `Message` output → **Prompt** → `question`
8. **Prompt** → `Prompt Message` output → **OpenAI** → `Input`
9. **OpenAI** → `Message` output → **Chat Output** → `Text`

> Chat Input connects to **two** things — Chroma DB (to search) and Prompt (as the question). That's correct.

When all 9 connections are made, every component should show no red warnings.

---

## Step 7 — Test in the Playground

1. Click the **Playground** button — top-right corner or bottom bar of the canvas
2. A chat window opens
3. Type a question about your document:

```
Who wrote the Meridian Protocol?
```

Expected answer: **Sandra Okafor**

Try a few more:
- `What happens if someone violates the protocol?`
- `When was the Meridian Protocol created?`
- `What day are status reports due?`

If the chatbot answers correctly from the document — the RAG pipeline is working.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Chatbot says "I don't know" for everything | Document not ingested | Click the **Run** (play) button on the Chroma DB node directly, then try Playground again |
| "Chroma connection refused" | Wrong host or containers not running | Confirm containers are up (`docker compose ps`), set host to `chromadb` and port to `8000` |
| Red borders on components | Missing required field | Click the red component and fill in the highlighted field |
| "API key invalid" error | Wrong key or extra spaces | Paste key again, no leading/trailing spaces |
| Connections won't snap | Type mismatch | Match dot colors — `Data` → `Data`, `Text` → `Text`, `Embeddings` → `Embeddings` |
| Prompt shows no `context` or `question` ports | Template not saved | Re-open Prompt, verify `{context}` and `{question}` are in the text, click Save |
| Generic answer not from document | Embedding model mismatch | Ensure the same `text-embedding-3-small` model is in the OpenAI Embeddings component |

---

## What's Happening Under the Hood

Each time you send a message in the Playground, LangFlow runs the entire flow:

1. **File** loads your document
2. **Split Text** cuts it into ~1000-character chunks with 200-char overlap
3. **OpenAI Embeddings** converts each chunk into a vector (a list of numbers)
4. **Chroma DB** stores all vectors — persisted to the `chroma_data/` Docker volume
5. **Chat Input** takes your question
6. **Chroma DB** embeds the question and finds the 4 most similar chunks
7. **Parse Data** formats those chunks as plain text
8. **Prompt** assembles: context chunks + your question into one message
9. **OpenAI** (`gpt-4o-mini`) reads the prompt and generates an answer
10. **Chat Output** displays the answer

---

## Saving Your Flow

- Click the **Save** icon (floppy disk) in the top bar — saves to LangFlow's internal storage in `langflow_data/`
- To export as JSON: click the **"..."** menu → **Export**
- To import on another machine: drag the JSON onto the canvas or use **Import Flow**
