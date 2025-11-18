# RAG (Retrieval-Augmented Generation) Documentation — AI Tender Query System

Nice — RAG will dramatically improve answer accuracy and reduce hallucinations. Below is a focused, developer-ready RAG design and implementation plan tailored to your AI Tender Query System (Mistral OCR + Fireworks LLM). It includes architecture, data pipeline, chunking, embedding & vector DB choices, retrieval logic, prompt templates, confidence scoring, fallback/escalation, endpoints, monitoring, and rollout steps.

---

# 1. Goals & high-level approach

**Goals**

- Ground LLM answers in tender text extracted by Mistral OCR.
- Retrieve only the most relevant passages (RAG) before calling the LLM.
- Provide citations (passage references / page numbers).
- Compute robust confidence & escalate when ambiguous.
- Keep latency & cost reasonable (embedding + vector DB caching).

**High-level flow**

1. OCR → plainText + layout metadata saved.
2. Preprocess: split tender into chunks (semantic + structural) and store embeddings in vector DB.
3. On question: encode question, retrieve top-k relevant chunks (hybrid scoring), optionally rerank, build context, call LLM with RAG prompt.
4. LLM returns answer + provenance + internal confidence.
5. Validate with verifier (similarity checks / secondary model). If low confidence → escalate to human.

---

# 2. Architecture & components

- **API / Backend** (Node.js/Express)

  - `/admin/upload` (existing)
  - `/admin/process-ocr/:tenderId` (existing)
  - **New**: `/admin/index-tender/:tenderId` → chunk + embed + upsert vector DB
  - **New**: `/bidder/ask` → RAG retrieval → LLM answer → validation → store

- **Storage**

  - Object store: S3/R2 for PDFs + raw OCR JSON
  - Relational/NoSQL DB: Metadata, questions, answers (MongoDB recommended)
  - Vector DB: Qdrant / Weaviate / Pinecone / Elasticsearch w/ vector plugin

- **Models**

  - Embeddings: `text-embedding-3-large` (or other high-quality model)
  - Generator: Fireworks `llama4-scout-instruct-basic` (your existing)
  - Optional reranker / verifier: smaller model or specialized reranker (e.g., Cohere reranker or a cross-encoder)

- **Workers**

  - Batch worker for indexing (chunk + embed)
  - On-demand worker for retrieval + LLM calls
  - Human-review queue worker for escalations

---

# 3. Data modeling & vector schema

**Vector DB document (per chunk)**

```json
{
  "id": "<tenderId>_<chunkIndex>",
  "tenderId": "<tenderId>",
  "chunkIndex": 12,
  "pageStart": 5,
  "pageEnd": 6,
  "section": "Eligibility", // optional if detected
  "text": "Full chunk text ...",
  "embedding": [
    /* float32 vector */
  ],
  "metadata": {
    "ocrRawUrl": "...",
    "pdfUrl": "...",
    "createdAt": "2025-11-18T12:34:56Z"
  }
}
```

**Relational/NoSQL additions**

- `Tenders` (existing)
- `OcrText` (existing)
- `Chunks` (optional table for quick reference)

  - chunkId, tenderId, chunkIndex, pageStart, pageEnd, textSummary, metadataUrl

- `Questions` (existing) — add fields:

  - `retrievedChunks: [{chunkId, score, snippet}]`
  - `contextUsed` (constructed prompt context)
  - `llmModel`, `llmParams`, `confidenceScore`, `provenance`

---

# 4. Chunking strategy

Good chunking is the single biggest influencer of RAG quality. Use hybrid structural + semantic chunking:

**Step A — Structural split**

- Use OCR layout metadata to split by:

  - Headings / section titles (if detected)
  - Page boundaries
  - Tables isolated as table-chunks

**Step B — Semantic chunking**

- For long sections, split into ~200–800 tokens (recommended 250–500 tokens best balance)
- Overlap windows by 20–30% to preserve context across boundaries

**Metadata to keep per chunk**

- tenderId, chunkIndex, pageStart/pageEnd, section heading, approximate tokenCount, short summary (first 150 chars)

**Algorithm (pseudo)**

1. parse OCR layout -> find headings
2. for each heading section:

   - concat text -> split into windows of N tokens (e.g., 400 tokens) with 100-token overlap

3. For tables: store as separate chunk with structured JSON if possible

---

# 5. Embedding & vector DB

**Embedding model**

- Use a semantically-strong embedding model (configurable):

  - placeholder: `text-embedding-3-large` or Fireworks embedding if available

- Batch embeddings for efficiency and caching.
- Normalize vectors (if DB requires) and use float32 arrays.

**Vector DB**

- Choose one: Qdrant, Pinecone, Weaviate, or Elasticsearch vector plugin.
- Index config:

  - distance metric: cosine (or dot) similarity
  - store metadata fields for provenance
  - support for hybrid retrieval (vector + metadata filters)

**Indexing pipeline**

1. When OCR processed -> chunk -> embed -> upsert into vector DB (idempotent).
2. Maintain `chunks` collection with same ids (for quick retrieval/display).

**Retention / pruning**

- If tender versions update, create new namespace/index or version tag.
- Provide TTL for ephemeral test docs.

---

# 6. Retrieval logic & ranking

**Hybrid retrieval (recommended)**

1. **Keyword filter**: Exact match metadata filters (e.g., section = "Eligibility", page range filter)
2. **Vector search**: Top-N by cosine similarity (k=15)
3. **Rerank**: Use a cross-encoder or simple semantic similarity re-score between question+chunk text to get final top-k (k=3–5)
4. **Aggregate evidence**: compute coverage score and token budget for LLM context

**Retrieval parameters (example)**

- initialVectorK = 15
- finalTopK = 4
- chunkTokenLimit = 1500 tokens total for context (balance with model limits)
- rerankerThreshold = 0.45 similarity (normalize per model)

**Scoring**

- `finalScore = alpha * vectorScore + beta * bm25Score + gamma * rerankerScore`

  - alpha ~ 0.6, beta ~ 0.2, gamma ~ 0.2 (tune on real data)

- Optionally use passage length penalty or redundancy penalty.

---

# 7. Prompting & context construction

**Prompt template (RAG) — canonical**

```
SYSTEM:
You are a helpful assistant specialized in answering questions about tender documents. Always cite the page numbers or chunk IDs used. If the answer is uncertain, state it and recommend escalation.

INSTRUCTIONS:
- Use only the provided document text (below) to answer.
- When quoting, wrap quotation in quotes and cite (page X).
- If the document doesn’t contain an answer, respond: "Not found in document — escalate to human."

DOCUMENT (sources follow):
Source 1: chunkId=<id1> pages=<p1>-<p2>
<chunk1 text>

Source 2: chunkId=<id2> pages=<p3>-<p4>
<chunk2 text>

QUESTION:
<user question>

RESPONSE FORMAT (JSON):
{
  "answer": "<answer text>",
  "evidence": [
    {"chunkId":"<id1>", "pageStart": <p1>, "snippet":"..."},
    ...
  ],
  "confidence": "<low|medium|high>",
  "shouldEscalate": <true|false>
}
```

**Notes**

- Provide both short direct answer and optional detailed rationale.
- Enforce output JSON to parse programmatically.
- Keep LLM token context limited — include only finalTopK reranked chunks.

---

# 8. Confidence scoring & verification

**Multi-step confidence check**

1. **LLM internal**: Ask generator to provide a numeric confidence estimate and evidence list (as part of JSON).
2. **Semantic validation**:

   - Compute semantic similarity between answer and retrieved chunks. If `sim(answer, topChunk)` < threshold → lower confidence.
   - Count how many chunks contributed (diversity).

3. **Hallucination checks**:

   - Use rule-based checks: dates in answer present in document? numeric values match?
   - Named Entity check: ensure named orgs/amounts match retrieval.

4. **Secondary verifier** (optional): send (question + answer + sources) to small verifier model to get binary PASS/FAIL.

**Confidence mapping**

- `high`: LLM confidence >= 0.85 AND semantic similarity >= 0.7 AND numeric checks passed.
- `medium`: LLM confidence 0.6–0.85 OR one minor check failed.
- `low`: LLM confidence < 0.6 OR major mismatch → escalate.

**Escalation rules**

- If classification is `LEAGAL_CLARIFICATION` or `NEGOTIATION`, prefer `ESCALATE_TO_HUMAN` unless confidence is very high.
- Always escalate on `UNKNOWN` or if evidence count = 0.

---

# 9. Caching, latency & cost management

**Cache layers**

- **Embeddings cache**: store embeddings for tender chunks (persistent).
- **Retrieval cache**: for repeated questions, cache top-k chunk IDs + LLM answer for X minutes (e.g., 30–60 min).
- **LLM call dedup**: detect duplicate questions to avoid re-calling generator.

**Batching & rate-limits**

- Batch embedding calls during indexing.
- Use concurrency limits for LLM calls; scale workers.

**Token budget**

- Request only top-k chunks and prioritize short but high-quality chunks.
- Use `answer_length` param to control LLM output tokens.

---

# 10. API & endpoints (detailed)

**1) Index tender (admin)**

- `POST /admin/index-tender/:tenderId`
- Steps:

  - Fetch OCR plainText + layout
  - Chunk & create chunk records
  - Generate embeddings (batch)
  - Upsert into vector DB

- Response: `{ status: "indexed", tenderId, chunkCount }`

**2) Ask (bidder)**

- `POST /bidder/ask`
- Body:

```json
{
  "tenderId": "T123",
  "bidderId": "B456",
  "question": "By what date must the security deposit be submitted?",
  "preferences": { "language": "en" }
}
```

- Flow:

  1. Classify queryType (existing classifier).
  2. Retrieval: hybrid search -> rerank -> topK.
  3. Build RAG prompt + call LLM.
  4. Verify confidence.
  5. Save result and return.

- Response:

```json
{
  "questionId": "Q789",
  "answer": "...",
  "classification": "DATE_ISSUE",
  "confidenceScore": 0.82,
  "status": "AI_ANSWERED",
  "evidence": [{ "chunkId": "...", "pageStart": 5, "snippet": "..." }]
}
```

**3) Re-index / update tender**

- `POST /admin/reindex-tender/:tenderId` with optional `version` to support versioning.

**4) Human review**

- `GET /admin/human-queue` — list escalations
- `POST /admin/human-answer/:questionId` — submit human-corrected answer, which should be fed back for training.

---

# 11. Human-in-the-loop & model training

**Human reviewer UI**

- Display: question, AI answer, retrieved chunks with highlights, confidence reasons, and an “accept / edit / escalate” workflow.
- Allow reviewer to mark which chunk(s) were actually useful (for future supervised fine-tuning).

**Feedback loop**

- Store (question, retrievedChunks, aiAnswer, humanAnswer, outcome)
- Periodically train a reranker or classifier with these labeled examples.
- Use these examples to tune chunking, retrieval weights, or fine-tune a local model.

---

# 12. Evaluation metrics & testing

**Offline metrics**

- Precision@k / Recall@k for retrieval (use held-out Q/A pairs).
- Answer F1 / Exact Match vs human answers.
- Reranker accuracy.

**Online metrics**

- Escalation rate (%) — target low but safe.
- Human override rate (how often humans modify AI answers).
- Time-to-answer / latency.
- User satisfaction rating (collect thumbs up/down).

**Automated tests**

- Unit tests for chunking and embedding pipeline.
- Integration tests for retrieval + LLM (with mocked models).
- Regression tests: ensure no increase in hallucination compared to baseline.

---

# 13. Security, compliance & privacy

- Encrypt embeddings & vectors if tender content is sensitive.
- Secure vector DB (VPC, private network).
- Ensure PDFs + OCR raw JSON stored with proper ACLs.
- Redact PII if bidder asks include personal data; store in secure db.
- Audit logs: record which chunks were used for every answer (for compliance).

---

# 14. Monitoring & observability

- Track metrics:

  - embedding time, retrieval latency, LLM latency, total time.
  - confidence distributions, escalation counts

- Alerts:

  - sudden spike in low-confidence answers
  - vector DB errors or timeouts

- Logs: store request/response with truncated text to avoid huge logs.

---

# 15. Rollout plan & incremental implementation

**Phase 0 — Minimal RAG**

- Implement chunking + embeddings + vector DB index for recent tenders.
- Implement `/bidder/ask` retrieval using vector search only (no reranker).
- Return AI answer with provenance (chunk ids).

**Phase 1 — Add reranker & verifier**

- Add cross-encoder reranker to improve top-k quality.
- Implement semantic validation & confidence thresholds.
- Add basic human review queue & UI.

**Phase 2 — Improve UX & scale**

- Add citations (page numbers) in answers.
- Add caching, batching, rate limits.
- Add training loop from human corrections (retrain reranker).

**Phase 3 — Advanced**

- Multi-language support, table extraction, vector compression, on-prem options.

---

# 16. Example Node.js snippets (pseudocode)

**Chunking + embedding & upsert**

```js
// pseudocode
const chunks = chunkTenderText(ocrPlainText, layoutMeta);
const texts = chunks.map((c) => c.text);
const embeddings = await embedModel.embedBatch(texts); // batch
const vectorRecords = chunks.map((c, i) => ({
  id: `${tenderId}_${c.index}`,
  vector: embeddings[i],
  payload: {
    tenderId,
    pageStart: c.pageStart,
    section: c.section,
    text: c.text.slice(0, 1000),
  },
}));
await vectorDb.upsert(vectorRecords);
await db.insertMany('chunks', chunksMetadata);
```

**Ask endpoint (simplified)**

```js
app.post('/bidder/ask', async (req, res) => {
  const { tenderId, question, bidderId } = req.body;
  const qEmbedding = await embedModel.embed(question);
  const hits = await vectorDb.search(qEmbedding, {
    topK: 15,
    filter: { tenderId },
  });
  const reranked = await reranker.rerank(question, hits); // optional
  const topK = reranked.slice(0, 4);
  const context = buildContext(topK);
  const prompt = createRagPrompt(context, question);
  const llmResp = await fireworks.generate({ prompt, maxTokens: 500 });
  const confidence = computeConfidence(llmResp, topK);
  const status = confidence < 0.6 ? 'ESCALATED' : 'AI_ANSWERED';
  const record = await db.insert('questions', {
    tenderId,
    bidderId,
    question,
    retrievedChunks: topK,
    aiAnswer: llmResp.text,
    confidence,
    status,
  });
  if (status === 'ESCALATED') enqueueHumanReview(record);
  res.json({ questionId: record.id, answer: llmResp.text, confidence, status });
});
```

---

# 17. Provenance & UI suggestions

- Show bidders: short answer + “Sources” section listing page numbers and excerpted snippets with links to the tender PDF.
- Allow users to click “Show full context” to see the chunk text.
- For admins/human reviewers: show similarity scores and confidence reasons.

---

# 18. Risks & mitigations

- **Hallucination risk** — mitigate with strict RAG prompt that instructs to only use given text; implement verifier checks and escalate.
- **Latency** — mitigate with caching, smaller reranker, limit topK.
- **Cost** — embeddings and LLM calls cost; batch embeddings, reuse embeddings, and cache top answers.
- **Legal content** — auto-escalate `LEAGAL_CLARIFICATION`.

---

# 19. Checklist to get started (actionable)

1. Choose vector DB & provision it (Qdrant recommended for self-host).
2. Implement chunking pipeline (structural + semantic).
3. Add embedding step & store embeddings in vector DB.
4. Implement `/admin/index-tender/:tenderId` and test indexing.
5. Implement retrieval-only RAG in `/bidder/ask` with top-k vector search and LLM prompt.
6. Add simple confidence heuristic and human queue for escalations.
7. Add reranker & verifier.
8. Add monitoring, metrics, and human feedback loop.
9. Tune thresholds using a labeled test dataset.

---

# 20. Helpful tuning tips

- Start with larger chunk sizes (400–600 tokens), measure retrieval precision, then adjust.
- Use overlap to avoid splitting key sentence across chunks.
- Keep k small for LLM context (3–5) and rely on reranker quality.
- For legal/financial categories, be conservative — prefer escalation.
- Log everything (for training) but minimize storage of full text in logs.

---
