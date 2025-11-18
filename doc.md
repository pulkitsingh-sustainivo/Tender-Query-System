# AI Tender Query System – Developer Documentation

## 1. Project Overview

A platform where admins upload tender PDF documents. The system extracts OCR text using **Mistral OCR (mistral-ocr-2505)**, stores data in the database & S3, and allows bidders to submit queries. Queries are classified into predefined categories, and responses are generated using **Fireworks AI – llama4-scout-instruct-basic**. If the query cannot be answered confidently, it is escalated to a human reviewer.

---

## 2. Features

### **Admin**

- Manage tender PDFs
- Trigger OCR processing
- Review bidder queries
- Handle escalated queries

### **Bidder**

- Ask tender-related questions
- Receive AI-generated responses

### **AI System**

- Extract OCR text
- Classify queries
- Generate responses
- Escalate when needed

---

## 3. Media Uploads

- PDFs uploaded via API endpoint
- Validate file format, size, and type
- Store file in **S3 bucket** under structured folder naming
- Create database entry with:

  - tenderId
  - file metadata
  - s3 URL
  - upload timestamp

---

## 4. OCR Processing (Mistral OCR – `mistral-ocr-2505`)

### Flow:

1. File uploaded to S3
2. Server sends PDF to **Mistral OCR API**
3. OCR returns:

   - **plain text**
   - **raw OCR JSON response** (tables, layout, metadata)

4. Save:

   - Plain text → **Database**
   - Raw JSON → **S3**

### Stored Fields:

- tenderId
- plainText
- ocrRawResponseUrl
- processedAt

---

## 5. Query Classification

Each bidder query must be classified into the following types:

1. FINANCIAL_CLARIFICATION
2. SPELLING_MISTAKE
3. CLARIFICATION
4. DATE_ISSUE
5. ELIGIBILITY_CRITERIA
6. PROCEDURAL
7. REQUIREMENTS
8. NEGOTIATION
9. ORDER_OR_WORK_ASSIGNMENT
10. GENERAL_INQUIRY
11. LEAGAL_CLARIFICATION
12. DOCUMENT_NAVIGATION
13. SUBMISSION_PROCESS
14. TERMS_AND_CONDITIONS
15. ESCALATE_TO_HUMAN
16. UNKNOWN

Classification is performed using an LLM-based prompt instruction.

## 6. Answer Generation (Fireworks AI – `llama4-scout-instruct-basic`)

### Steps:

1. Retrieve tender OCR text from database
2. Run a relevance filter (optional)
3. Generate answer using Fireworks LLM
4. If confidence score is low → set classification to **ESCALATE_TO_HUMAN**

### Stored Fields:

- question
- classification
- aiAnswer
- confidenceScore
- status: AI_ANSWERED, ESCALATED, HUMAN_ANSWERED
- timestamps

---

## 7. Storage Structure

### **S3 Bucket**

- `/tenders/{tenderId}/original.pdf`
- `/tenders/{tenderId}/ocr/raw.json`

### **Database** (MongoDB or PostgreSQL)

#### Tenders

- tenderId
- title
- pdfUrl
- createdBy
- createdAt

#### OCRText

- tenderId
- plainText
- rawOcrUrl
- processedAt

#### Questions

- questionId
- tenderId
- bidderId
- question
- classification
- answer
- status
- createdAt

---

## 8. API Flow

### 1. Admin uploads tender PDF

`POST /admin/upload`

- saves file → S3
- returns tenderId

### 2. OCR processing

`POST /admin/process-ocr/:tenderId`

- sends PDF → Mistral OCR
- save response → DB + S3

### 3. Bidder asks question

`POST /bidder/ask`

- classify query
- generate answer
- if not answerable → escalate

---

## 9. Architecture Diagram
flowchart TD

    B(Bidder) -->|Ask Question| QAPI[Backend API<br>/ask-question]
    A(Admin) -->|Upload Tender PDFs| UAPI[Backend API<br>/upload-tender]

    UAPI --> S3[(S3 Storage)]
    UAPI --> OCR[Mistral OCR<br>mistral-ocr-2505]
    OCR --> TEXT[Extracted Plain Text]
    OCR --> RAW[Raw JSON OCR Output]
    TEXT --> DB[(Database)]
    RAW --> S3

    QAPI --> CLASSIFY[Query Classifier<br>(LLM-based)]
    CLASSIFY --> ROUTE{AI Answerable?}

    ROUTE -->|Yes| FIREWORKS[Fireworks AI<br>llama4-scout-instruct-basic]
    FIREWORKS --> ANSWER[AI Answer]
    ANSWER --> QAPI

    ROUTE -->|No| HUMAN[Human Reviewer]
    HUMAN --> QAPI

---

## 10. Developer Setup

### Requirements

- Node.js 18+
- MongoDB / PostgreSQL
- AWS S3 / Cloudflare R2
- API Keys:

  - Mistral OCR
  - Fireworks AI

### Steps

1. Clone repo
2. Add environment variables
3. Run DB migrations
4. Start server

---

## 11. Error Handling

- PDF upload failures
- OCR API timeout
- Empty OCR extraction
- AI response failure
- Classification fallback to UNKNOWN
- Escalation flow

---

## 12. Future Enhancements

- Add semantic vector search
- Add rate limiting for bidder queries
- Improve answer accuracy with RAG model
- Multi-language OCR

---

Here is an expanded and enhanced **Future Enhancements** section with deeper technical detail, architecture upgrades, and planned next-gen improvements. You can drop this directly into your documentation.

---

## **12. Future Enhancements (Expanded & Detailed)**

The following improvements are planned to make the AI Tender Query System more accurate, scalable, and feature-rich.

---

### **12.1 Advanced Semantic Search (Vector + Hybrid Search)**

Move beyond plain-text search by integrating vector-based semantic retrieval.

**Planned Features:**

* Embed all OCR text using models such as **OpenAI text-embedding-3-large**, **Mistral embeddings**, or **Fireworks embeddings**
* Store embeddings in a vector DB (Pinecone, Weaviate, Qdrant, Elasticsearch w/ vector plugin)
* Hybrid retrieval: **keyword + semantic**
* Reduce irrelevant context passed to LLM
* Improve accuracy for long queries or ambiguous questions

**Benefits:**

* Faster retrieval
* Higher answer precision
* Less hallucination from LLM due to improved grounding

---

### **12.2 Improved RAG (Retrieval-Augmented Generation)**

Enhance answer generation using more structured retrieval.

**Proposed Additions:**

* Chunk tender text intelligently (page-level, section-level, semantic chunks)
* Use Mistral OCR layout metadata (tables, headings) to maintain structure
* Add citation references in AI-generated answers
* Multi-step reasoning with chain-of-thought suppression
* Use “rerankers” like Cohere Rerank or BAAI-bge-reranker

**Benefits:**

* Answers become more trustworthy
* Transparency increases (answer shows which parts of document were used)

---

### **12.3 Better Confidence Scoring & AI Safety Layer**

Currently, escalation is based on simple confidence heuristics. We plan to enhance this with:

**Enhancements:**

* Dual-model validation: one model generates answer, another verifies accuracy
* Semantic similarity scoring between question, retrieved context, and answer
* Hallucination detection using specialized models
* Rule-based safety checks (missing data, ambiguous questions, legal content)
* Automatic escalation triggers when uncertainty is detected

**Benefits:**

* Reduced risk of incorrect AI answers
* More reliable classification into "ESCALATE_TO_HUMAN"

---

### **12.4 Multi-Language Support**

Enable OCR + answering in multiple languages.

**Planned:**

* Support Hindi, Tamil, Bengali, Telugu, Sinhalese, Arabic, French, etc.
* Use multilingual models for classification and generation
* Language autodetection in OCR and question inputs

**Benefits:**

* Expand system to international tenders
* Improve accessibility

---

### **12.5 Bidder Portal Enhancements**

Add user-friendly tools for bidders.

**Features:**

* View previously asked questions + historical answers
* Suggested questions based on tender sections
* Inline “Ask AI” button for each section of the document
* AI-powered Tender Summary

**Benefits:**

* Simplifies bidder experience
* Reduces repetitive or duplicate queries

---

### **12.6 Admin Dashboard Improvements**

More powerful analytics and workflow automation.

**Planned:**

* Human review queue with priority scoring
* AI vs Human performance analytics
* Query category analytics
* Tender completeness indicator (OCR quality, missing pages, etc.)
* Export Q&A logs to CSV/PDF

---

### **12.7 Processing Queue & Auto-Scaling**

Introduce background processing at scale.

**Enhancements:**

* Message queues (RabbitMQ, AWS SQS, or Kafka) for OCR and classifier tasks
* Auto-scaling worker nodes for OCR-heavy workloads
* Rate limiting per user or tender
* Batch processing for large tenders

**Benefits:**

* High reliability under load
* Reduced downtime
* Smooth handling of large tenders (hundreds of pages)

---

### **12.8 Structured Data Extraction**

Beyond plain OCR text.

**Features:**

* Extract tables in clean CSV or JSON format
* Detect key sections (Eligibility, Criteria, Financial Terms, Submission Process)
* Auto-generate metadata fields for tender indexing
* Named Entity Recognition for:

  * dates
  * fees
  * deadlines
  * agencies
  * contact info

**Benefits:**

* Improved searchability
* Better contextual answering
* More automation for admins

---

### **12.9 Plugin Architecture for New OCR Providers & LLM Models**

Make the system model-agnostic.

**Planned:**

* Support switching between OCR engines (Mistral OCR, Tesseract, Google OCR)
* Add plug-in support for different LLMs
* Fallback model usage if a provider fails
* A/B testing of AI providers

**Benefits:**

* Future-proof design
* Cost optimization
* Ability to leverage specialized models for certain tasks

---

### **12.10 Real-Time Chat Mode for Bidders**

Instead of one-off questions, enable chat-with-tender mode.

**Enhancements:**

* Memory of past messages in a session
* Use RAG for continuous conversations
* Highlight relevant sections dynamically
* Provide citations inline as “references”

---

### **12.11 Smart Alerts & Deadlines Monitoring**

Automatically track tender deadlines and changes.

**Features:**

* Deadline reminders
* Change detection on updated tender documents
* Alert bidders or admins when important data changes

---

### **12.12 Security, Audit Logging & Compliance**

Strengthen enterprise-grade reliability.

**Planned:**

* Full audit logging for every AI decision
* Track which data was retrieved for each answer
* Tender data encryption at-rest and in-transit
* Hardened API security (JWT rotation, HMAC signatures)

---

### **12.13 Versioned Tender Documents**

Support multiple versions of the same tender.

**Benefits:**

* Track changes between old vs new tenders
* Show diff and update OCR records
* Ensure answers use the most recent version

---

### **12.14 Offline or On-Prem Deployment Options**

For enterprise/government clients.

**Possible Options:**

* Deploy on private cloud
* Run LLMs on local GPU clusters
* Data never leaves organization

---

### **12.15 AI Training on Repeated Escalations**

Improve the system with fine-tuning.

**Plan:**

* Collect frequently escalated queries
* Train custom mini-models
* Improve classifier accuracy
* Reduce escalations over time

---

