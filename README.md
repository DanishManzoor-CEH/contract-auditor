# 📑 Smart Multi-Document Compliance & Contract Auditor

An AI-driven web app for legal, finance, and compliance teams. Upload multiple
contracts or policy PDFs, ask questions across all of them in plain English,
and get answers that are **grounded in your documents, cited to the exact
page, and independently fact-checked** before you see them.

## Why this exists

Compliance officers and legal auditors routinely burn hours scanning 50+ page
contracts for a single clause. Generic AI chatbots make this worse, not
better — they hallucinate confident-sounding answers or rely on outdated
general knowledge instead of your actual documents. This app restricts the AI
to only answer from text retrieved directly from what you uploaded, and adds
a second AI pass whose only job is to check the first answer for
unsupported claims.

## How it works (architecture)

**Ingestion path** (runs once per uploaded batch):

```
PDF(s) --> pypdf text extraction --> RecursiveCharacterTextSplitter
        --> Sentence-Transformer embeddings --> FAISS vector index
```

**Query path** (runs once per question):

```
User question
   │
   ▼
Embed question (Sentence-Transformers)
   │
   ▼
FAISS similarity search  ──►  Top-K matching chunks (with doc + page labels)
   │
   ▼
Groq LLM: generate answer strictly from those chunks
   │
   ▼
Groq LLM: Corrective RAG validation
   (does the context actually support this answer? what's missing?)
   │
   ▼
Streamlit UI: answer + validation verdict + expandable cited source text
```

Because chunking keeps track of `(source filename, page number)` for every
piece of text, every answer can be traced back to the exact paragraph it came
from — click "Show cited source chunk(s)" in the UI to see it.

## Tech stack

| Layer               | Tool                                   |
|---------------------|-----------------------------------------|
| PDF parsing          | `pypdf`                                |
| Chunking             | `langchain-text-splitters` (RecursiveCharacterTextSplitter) |
| Embeddings           | `sentence-transformers` (`all-MiniLM-L6-v2`) |
| Vector search        | `faiss-cpu`                            |
| LLM generation + validation | `groq` (free-tier API)          |
| UI                   | `streamlit`                            |
| Dev environment      | Google Colab (T4 GPU)                  |
| Hosting              | Streamlit Community Cloud + GitHub     |

## Project structure

```
contract-auditor/
├── app.py                          # Main Streamlit application
├── requirements.txt                # Pinned dependencies
├── README.md                       # This file
├── .gitignore
├── .streamlit/
│   └── secrets.toml.example        # Template — copy to secrets.toml locally
└── notebooks/
    └── colab_prototype.ipynb       # Interactive prototyping notebook
```

---

## 1. Prototyping in Google Colab (T4 GPU)

You don't need Colab to run the final app, but it's the fastest place to
sanity-check the pipeline (chunking, embeddings, FAISS search) before
wrapping it in Streamlit — especially the first time you try a new PDF or
tweak chunk size.

1. Go to [colab.research.google.com](https://colab.research.google.com) and
   create a new notebook (or open `notebooks/colab_prototype.ipynb` from this
   repo via File → Upload notebook).
2. Turn on the GPU runtime: **Runtime → Change runtime type → Hardware
   accelerator → T4 GPU → Save**. (The embedding model runs fine on CPU too,
   but GPU speeds up encoding many chunks/documents at once.)
3. In the first cell, install dependencies:
   ```python
   !pip install pypdf langchain-text-splitters sentence-transformers faiss-cpu groq
   ```
4. Upload a sample PDF using the Colab file browser (folder icon in the left
   sidebar → upload) or `from google.colab import files; files.upload()`.
5. Follow the cells in `notebooks/colab_prototype.ipynb` to: extract text,
   chunk it, embed it, build a FAISS index, run a similarity search, and call
   the Groq API for an answer. This mirrors exactly what `app.py` does, just
   without the Streamlit UI wrapper, so it's easier to inspect intermediate
   outputs (chunk boundaries, retrieved passages, raw model output).
6. Get a free Groq API key at [console.groq.com/keys](https://console.groq.com/keys)
   and set it in Colab via `Secrets` (key icon in left sidebar) rather than
   pasting it into a cell.

Once the pipeline behaves the way you want in the notebook, move on to
running the real app locally.

## 2. Running locally

```bash
git clone https://github.com/<your-username>/contract-auditor.git
cd contract-auditor
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Set your Groq API key locally:

```bash
cp .streamlit/secrets.toml.example .streamlit/secrets.toml
# then edit .streamlit/secrets.toml and paste your real key
```

Run the app:

```bash
streamlit run app.py
```

Open the URL Streamlit prints (usually `http://localhost:8501`), upload a
test PDF, click **Process & Index Documents**, then ask a question.

## 3. Pushing to GitHub

```bash
git init
git add .
git commit -m "Initial commit: Smart Multi-Document Compliance & Contract Auditor"
git branch -M main
git remote add origin https://github.com/<your-username>/contract-auditor.git
git push -u origin main
```

`secrets.toml` is listed in `.gitignore` and will **not** be pushed — only
`secrets.toml.example` (with a placeholder, not your real key) goes to
GitHub.

## 4. Deploying to Streamlit Community Cloud

See the full numbered checklist below in **Deployment Guide**.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "GROQ_API_KEY not found" error in the app | Secret not set | Add it in `.streamlit/secrets.toml` (local) or the Cloud Secrets Manager (deployed) |
| App crashes on deploy with a memory error | `sentence-transformers`/`torch` are large; free tier has limited RAM | Keep to the pinned `all-MiniLM-L6-v2` model (small); avoid larger embedding models |
| "No text could be extracted" for a PDF | The PDF is scanned/image-only with no text layer | Run it through OCR (e.g. `ocrmypdf`) before uploading |
| Slow first query after deploy | Cold start — embedding model is downloading/loading | Expected once per app restart; subsequent queries are fast |
| Groq API errors (rate limit / auth) | Free-tier rate limits or invalid key | Check [console.groq.com](https://console.groq.com) for usage/limits and confirm the key is active |
