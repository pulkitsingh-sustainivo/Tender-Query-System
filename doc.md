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

```mermaid
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
```

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
