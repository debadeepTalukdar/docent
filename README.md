# Docent: a docs agent that proves its answers

Ask questions about any documentation folder. Docent retrieves, writes code,
**runs the code to check it works**, and cites sources with **quotes that are
verified word for word** against the index. A CI gate fails the build if search quality drops.

## Why it exists
Docs bots hallucinate APIs and invent citations. Docent makes both checkable:
- `code_verified` is computed by the server (did a snippet actually exit 0?), not claimed by the model.
- A citation only counts if its chunk was retrieved and its quote appears verbatim in it.
  One retry is allowed, then bad citations are dropped and `grounded` is false.

## Architecture
```
question -> agent loop (max 8 steps)
  |- search_docs: dense cosine + SQLite FTS5 BM25 -> Reciprocal Rank Fusion
  |- read_chunk, run_python (sandboxed subprocess)
  `- submit_answer -> Pydantic schema -> citation verifier -> Answer
```
Index: heading-aware markdown chunking (code fences never split), content-hashed
incremental sync (only changed chunks re-embedded, deleted ones removed), `POST /reindex` for webhooks.

## Run locally (no Docker)
```bash
pip install -r requirements.txt
export ANTHROPIC_API_KEY=your_key
export ENABLE_CODE_EXEC=1          # only on a machine you trust
uvicorn app.main:app --reload
```
```bash
curl -X POST localhost:8000/ask -H 'content-type: application/json' \
  -d '{"session_id":"me","question":"How do I avoid SQL injection when inserting values?"}'
```

## Run with Docker
```bash
cp .env.example .env     # add your ANTHROPIC_API_KEY
docker compose up --build
```
Index your own docs: mount them in `docker-compose.yml`, set `DOCS_DIR=/docs`, then `POST /reindex` after changes.

## Tests and evals
```bash
pytest -q                        # unit + end-to-end tests (fake embedder, scripted fake LLM)
python -m evals.run              # hit@5 and MRR: vector vs BM25 vs hybrid (no API key needed)
python -m evals.gate             # exits 1 if a metric falls below evals/thresholds.json
python -m evals.run --agent 6    # grounded / gold-cited / code-verified rates + LLM judge
python -m evals.generate 30      # LLM-written questions to grow the eval set (review them!)
```

## Honest limits
- Vector search is brute-force numpy: fine for ~50k chunks, then move to pgvector/FAISS.
- The sandbox limits time, CPU and memory but is not a security boundary. Production needs gVisor/Firecracker or an isolated container. Code execution is off by default.
- The shipped eval set is tiny (6 questions, 2 files); the numbers mean something only on a real docs corpus.
