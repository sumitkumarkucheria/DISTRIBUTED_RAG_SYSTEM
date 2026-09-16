# Distributed RAG System

A local, privacy-preserving Retrieval-Augmented Generation (RAG) system that answers questions strictly from your own documents — no data ever leaves your network, and no cloud LLM API is used. Built to run entirely on CPU across a small LAN cluster of Ollama-backed worker machines.


Most RAG tutorials assume a single machine and a hosted LLM API. This system is designed for a different constraint set: **keep documents and inference entirely on-premises** (useful for private/company data), and **make use of multiple ordinary CPU-only machines** instead of one GPU box. It load-balances inference requests across whichever worker machines are online, and degrades gracefully if some are unreachable.

Architecture


                    
   documents(.pdf / .txt)/  ───▶    build_index.py     → embeds & chunks docs (BGE-small) →  FAISS index + chunk store
                                
                                
                     
   client query ───▶ retrieval_server.py <-- FastAPI: /search, /ask, /stream_ask
                      (FAISS + BGE)       
                     
            retrieval_server.py routes to least-busy worker   
               worker 1 (Ollama, qwen2.5)    
               worker 2 (Ollama, qwen2.5)    
               worker 3 (Ollama, qwen2.5)   
               worker 4 (Ollama, qwen2.5)
               Add as many workers as you need.
                   


- **Embedding & indexing** (build_index.py) — reads PDFs (pdfplumber) and text files, chunks them (500 chars, 100-char overlap), embeds with BAAI/bge-small-en-v1.5, and builds a FAISS IndexFlatL2 index.
- **Retrieval + generation server** (retrieval_server.py) — a FastAPI service exposing:
  - GET /search — raw nearest-neighbor chunk retrieval, useful for debugging relevance.
  - POST /ask — full RAG: retrieves top-5 chunks, enforces a **distance threshold** so the system explicitly refuses to answer when nothing relevant is found (rather than hallucinating), and generates via the least-loaded worker.
  - POST /stream_ask — same as /ask, but token-streamed back to the client.
- **Load balancing** — get_least_busy_worker() pings each configured worker and routes to whichever has the lowest current in-flight request count, so a slow worker doesn't become a bottleneck for the whole cluster.
- **Workers** — plain [Ollama](https://ollama.com) instances running qwen2.5:1.5b on each LAN machine; no GPU required.

## Why it doesn't hallucinate outside its documents

The system computes a similarity distance for the best-matching chunk on every query. If that distance exceeds a threshold (THRESHOLD = 1.20{You may change according to you document information chunks } ), it returns "Information not found in knowledge base" instead of asking the LLM to answer from its own (unverifiable) knowledge. This is a deliberate design choice — the tradeoff is a stricter, more conservative system that will decline answerable-but-borderline questions in exchange for not making things up.

## Getting started

bash
pip install fastapi sentence-transformers faiss-cpu pdfplumber numpy requests uvicorn

# 1. Drop your .pdf / .txt files into a documents/ folder, then build the index
python scripts/build_index.py

# 2. Edit the WORKERS list in scripts/retrieval_server.py to your LAN IPs,
#    and make sure Ollama + qwen2.5:1.5b are running on each one

# 3. Start the server
uvicorn scripts.retrieval_server:app --host 0.0.0.0 --port 8000 {Or you can have a secure connection too}


Then query it:

bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is photosynthesis?"}'


## Status / known limitations

- Paths are currently hardcoded to a Windows machine (D:\AI_SERVER\...) — needs to move to config/env vars before this is portable or shareable as a template.
- No requirements.txt yet — dependencies above are inferred from imports.
- Chunking is fixed-size character-based; sentence/semantic-aware chunking would likely improve retrieval quality.
- No authentication on the API — fine for a trusted LAN, not for anything exposed further.

## Roadmap

- [ ] Externalize paths and worker list into a config file / .env
- [ ] Add requirements.txt / pyproject.toml
- [ ] Swap fixed-size chunking for recursive/semantic chunking
- [ ] Add a minimal auth layer if ever exposed beyond localhost/LAN
