# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

FinanceBot is a RAG-powered AI chatbot for personal finance literacy. It consists of:
- `app.py` — Streamlit chatbot backend with Groq LLM integration
- `index.html` — Static landing page that embeds the chatbot via iframe
- `finance_kb.txt` — Knowledge base file (Q&A format, double-newline-separated chunks)

## Running the App

```powershell
# Install dependencies (see note below about requirements.txt)
pip install streamlit groq sentence-transformers scikit-learn numpy python-dotenv

# Run the chatbot
streamlit run app.py

# Run on a specific port (index.html embeds it at port 8502)
streamlit run app.py --server.port 8502
```

> **Note:** `requirements.txt` is outdated — it lists `openai` which is no longer used. The actual runtime dependencies are `streamlit`, `groq`, `sentence-transformers`, `scikit-learn`, `numpy`, and `python-dotenv`.

## API Key Setup

The app checks for `GROQ_API_KEY` in two places (in order):
1. `.env` file in the project root (`GROQ_API_KEY=your_key`)
2. `.streamlit/secrets.toml` — copy `.streamlit/secrets.toml.example` and fill in the key

Get a free API key at https://console.groq.com.

## Architecture

### RAG Pipeline (`app.py`)

Three main functions form the pipeline:

1. **`load_knowledge_base()`** — Reads `finance_kb.txt`, splits on double newlines into chunks, and encodes them with `sentence_transformers` (`all-MiniLM-L6-v2`). Wrapped in `@st.cache_resource` so it only runs once per session. Returns `(chunks, embeddings, embedder)`.

2. **`retrieve_context(query, chunks, ...)`** — Despite loading vector embeddings, retrieval is currently implemented as **keyword frequency matching** (counts how many query words appear in each chunk), not cosine similarity. Returns top-3 chunks joined as context.

3. **`get_groq_response(user_message, context, chat_history)`** — Calls Groq API with model `llama-3.3-70b-versatile`. Injects retrieved context into the system prompt. Keeps the last 6 messages of chat history for conversational context.

### Website Integration (`index.html`)

The landing page embeds the Streamlit app as an iframe. For local development it points to `http://localhost:8502`. When deployed to Streamlit Cloud, a JS snippet swaps the iframe `src` to `https://financebot-app.streamlit.app`. Update this URL in the script tag after deploying.

### Knowledge Base (`finance_kb.txt`)

Plain-text file with Q&A pairs separated by blank lines. Each paragraph becomes one retrieval chunk. Expand this file to improve bot coverage — no code changes needed.

## Deployment (Streamlit Cloud)

1. Push to GitHub, go to https://streamlit.io/cloud
2. New app → select repo → set main file to `app.py`
3. Under **Advanced settings → Secrets**, add: `GROQ_API_KEY = "your_key_here"`
4. After deploy, update the iframe URL in `index.html`

## Key Constraints

- `finance_bot_demo/` contains only a leftover `venv` — it is not a separate app
- `versions.txt` and `err.txt`/`out.txt` are diagnostic artifacts (gitignored)
- Chat history is capped at the last 6 messages sent to Groq to control token usage
- LLM temperature is set to `0.3` and `max_tokens` to `600` for concise, consistent answers
