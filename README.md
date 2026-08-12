[README.md](https://github.com/user-attachments/files/30964034/README.md)
# Cost-Efficient RAG Application

RAG service using persistent ChromaDB, `sentence-transformers/all-MiniLM-L6-v2` (384 dimensions), FastAPI, and OpenAI or Ollama.

## Defaults
- chunk size: 700 characters
- overlap: 120 characters
- top-k: 5
- minimum similarity: 0.25
- formats: PDF, HTML, MD

## Run
```bash
python -m venv .venv
.venv\\Scripts\\activate
pip install -r requirements.txt
copy .env.example .env
python scripts/ingest.py data/docs
uvicorn app.main:app --reload
```
Swagger: http://127.0.0.1:8000/docs

Query example:
```json
{"question":"What is the refund policy?","k":5,"filter":{"source_type":"pdf"}}
```

## Idempotency
A stable hash of the source path is the document ID. Re-ingestion deletes the existing document's chunks before inserting new ones, so vectors are replaced rather than duplicated.

## Evaluation
Replace `eval/questions.jsonl` with 15–30 fixed corpus-specific questions and label the relevant chunk IDs. Then:
```bash
python -m eval.run_eval --dataset eval/questions.jsonl --output eval/results.json
python -m eval.judge --results eval/results.json --output eval/results_judged.json
```
Metrics: Hit Rate@k, Recall@k, MRR, nDCG@k, Context Precision@k, EM/F1, retrieval p50/p95, end-to-end p50/p95, and LLM-judge faithfulness/relevance.

Do not report fabricated evaluation results; commit the generated results from the actual corpus run.

## Cost benchmark assumptions
A 384-d float32 vector is ~1.5 KB before index/metadata overhead. Raw payload is ~0.15 GB / 1.54 GB / 15.36 GB at 100K / 1M / 10M vectors.

For the assignment's always-on pod baseline, Pinecone's published pod pricing lists p1 at about 1M vectors and $0.111/hour, and s1 at about 5M vectors and $0.111/hour. At 730 hours/month:

| vectors | managed assumption | managed DB/mo | Chroma assumption |
|---:|---|---:|---:|
| 100K | 1 p1 | $81.03 | $10 |
| 1M | 1 p1 | $81.03 | $20 |
| 10M | 2 s1 | $162.06 | $60 |

These are benchmark assumptions, excluding LLM, embedding, app compute, egress and backup. Pinecone also has serverless usage-based pricing, so production comparison should use the exact current plan/workload.

## Switch back to managed when
You need HA/failover, multi-replica concurrent writes, managed backups/SLAs, larger-than-single-node scale, very high sustained QPS, or compliance controls that justify the operational premium.
