# PDF Agreement Explainer — Multi-Document RAG

A Python application that lets users upload multiple PDFs, search their contents semantically, and ask questions answered from retrieved document passages with page-level citations.

**Author:** Madhav Kumar

## Overview

Reading and comparing long PDF documents can be time-consuming, especially when relevant information is spread across several files. This application combines PDF text extraction, local text embeddings, vector search and a language model to help users find relevant passages and ask evidence-grounded questions.

Although the project can be used with agreements, the implementation is a general-purpose multi-document PDF search and question-answering application. It does not currently include agreement-specific clause extraction or legal advice functionality.

## Features

- **Multi-PDF upload:** Upload and index multiple text-based PDF files in one session.
- **PDF validation:** Check file extension, PDF signature, file size, corruption, encryption and empty documents.
- **Scanned-PDF detection:** Identify PDFs that do not contain enough embedded text and provide feedback when OCR is needed.
- **Page-aware extraction:** Extract text page by page and retain one-based page numbers.
- **Overlapping chunking:** Split page text into overlapping chunks without crossing page boundaries.
- **Local embeddings:** Generate normalized text embeddings with Sentence Transformers using `all-MiniLM-L6-v2` by default.
- **Vector search:** Store and retrieve document chunks with ChromaDB using cosine distance.
- **Cross-document retrieval:** Balance retrieved passages across matching documents to reduce the chance that one document dominates the context.
- **Scope filtering:** Search all indexed documents or restrict retrieval to selected files.
- **Grounded question answering:** Send retrieved evidence to the DeepSeek chat API with instructions to avoid unsupported answers and cite factual claims.
- **Page-level citations:** Display source filenames and page numbers alongside retrieved evidence.
- **Insufficient-evidence fallback:** Return a fixed response when retrieval does not find sufficiently relevant evidence, without calling the language model.
- **Document explorer:** Inspect indexed document metadata and extracted chunks.
- **Session isolation:** Keep document vectors in an ephemeral, session-scoped ChromaDB collection.

## Technology Stack

| Area | Technologies |
|---|---|
| Language | Python |
| User interface | Streamlit |
| PDF processing | PyMuPDF |
| Embeddings | Sentence Transformers |
| Vector database | ChromaDB |
| LLM integration | OpenAI Python SDK configured for the DeepSeek API |
| Configuration | python-dotenv |
| Testing | pytest |

## Architecture

The application is separated into three primary modules:

```text
                 Streamlit UI (app.py)
                         |
             Upload PDFs / Ask questions
                         |
              PDF ingestion (ingest.py)
              |  Validate file
              |  Extract text by page
              |  Create overlapping chunks
                         |
              Sentence Transformers
                 Local embeddings
                         |
              ChromaDB session store
                         |
          Semantic retrieval and filtering
                         |
             Evidence threshold check
                    /        \
             Not enough     Sufficient
              evidence      evidence
                 |              |
          Fixed fallback    DeepSeek API
                                 |
                       Answer with citations
```

### Module responsibilities

- **`app.py`:** Implements the Streamlit interface, session state, PDF upload and document management, chat history, multi-file search and document exploration.
- **`ingest.py`:** Validates uploaded PDFs, extracts page text, detects likely scanned documents and creates page-aware overlapping chunks.
- **`rag.py`:** Generates embeddings, manages the ChromaDB collection, retrieves relevant chunks, formats evidence and calls the DeepSeek-compatible chat API.

## PDF Ingestion and Chunking

Each uploaded file is validated before processing. The ingestion module checks the filename, PDF signature, upload size and whether the file can be opened. Password-protected and empty PDFs are rejected. Files with insufficient embedded text are identified as likely scanned PDFs; OCR is not performed by the application.

Text is extracted separately from every page. The chunking process uses a default chunk size of 1,200 characters and an overlap of 200 characters. Chunks retain the source filename, page number, document identifier and chunk index. Keeping chunks within individual pages ensures that a retrieved passage can be traced to a specific PDF page.

A SHA-256 hash of the uploaded PDF bytes is used as its document identifier.

## Retrieval and Question Answering

1. The embedding model converts each document chunk into a normalized vector.
2. ChromaDB stores vectors and metadata in a session-specific collection configured for cosine distance.
3. A user query is embedded using the same model.
4. The retriever obtains candidate passages and applies a distance threshold (default maximum distance: 0.75 for question answering).
5. For multi-document queries, the retriever first selects a strong candidate from each represented document, then fills remaining result slots by relevance.
6. If no passage satisfies the threshold, the application returns:

   `I don't have enough evidence in the uploaded PDFs to answer that.`

7. If relevant evidence exists, the application sends the question and retrieved passages to the DeepSeek API. The prompt instructs the model to answer only from the supplied evidence, report conflicting source information and cite factual claims using the source filename and page number.

The Search Across Files interface also supports direct semantic retrieval without calling the language model. Users can adjust the result limit, similarity threshold and document scope.

## Session and Data Handling

The application creates a unique session identifier and uses it when creating the ChromaDB collection. The vector store is ephemeral and scoped to the Streamlit session. Uploaded PDF bytes are processed in memory by the ingestion code and are not intentionally written to a project data directory.

The Sentence Transformers model is cached by Streamlit as a shared model resource; uploaded document content and vectors are maintained separately in session state and the session-scoped vector store.

## Getting Started

### Prerequisites

- Python 3.10, 3.11 or 3.12
- pip
- Internet access on first launch to download the embedding model and to call the configured language-model API
- A DeepSeek API key

### 1. Clone the repository

```bash
git clone https://github.com/yomadhavvvv/PDF-Agreement-Explainer.git
cd PDF-Agreement-Explainer/pdf-rag-app-main/pdf-rag-app-main
```

### 2. Create a virtual environment

**Windows PowerShell**

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

**macOS / Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### 4. Configure environment variables

Create a file named `.env` in the same directory as `app.py`:

```env
DEEPSEEK_API_KEY=your_api_key
DEEPSEEK_MODEL=deepseek-v4-flash
DEEPSEEK_BASE_URL=https://api.deepseek.com
EMBEDDING_MODEL=all-MiniLM-L6-v2
```

Only `DEEPSEEK_API_KEY` is required. The other variables have defaults in the application. Replace the placeholder with your own key. Never commit the `.env` file or expose API credentials in source code.

### 5. Run the application

```bash
python -m streamlit run app.py
```

Streamlit prints a local address in the terminal. Open it in a browser to use the application.

## Using the Application

1. Upload one or more text-based PDF documents from the sidebar.
2. Select **Process Selected PDFs** to validate, extract, chunk and index them.
3. Use **Multi-File Question Answering** to ask a question across the selected documents.
4. Use **Search Across Files** to find semantically relevant passages without generating an LLM response.
5. Use the document overview to inspect page counts, chunk counts and extracted text.
6. Remove individual documents or clear the session's indexed documents when finished.

Each uploaded file is limited to 25 MB by the ingestion module.

## Tests

The repository includes pytest tests for the ingestion and retrieval modules. The tests use generated PDF files and a deterministic fake embedding model so that core behaviors can be tested without downloading the Sentence Transformers model or making live language-model API calls.

The tests cover:

- PDF filename, signature and empty-file validation.
- Detection of scanned/image-only PDFs.
- Page number preservation and page-local chunking.
- Chunk overlap and parameter validation.
- Multiple PDF ingestion with partial failures.
- Adding, retrieving and deleting document vectors.
- Similarity-distance filtering and document-specific retrieval.
- Balanced retrieval across multiple documents.
- Insufficient-evidence handling.
- DeepSeek request construction using a fake client.
- Recovery from a problematic Chroma ephemeral-client cache state.

Run the tests from the project directory:

```bash
python -m pytest -q
```

The test suite does not require a real DeepSeek API key. It does require the Python dependencies in `requirements.txt`.

## Configuration

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `DEEPSEEK_API_KEY` | Yes for generated answers | None | API credential for the DeepSeek-compatible chat endpoint |
| `DEEPSEEK_MODEL` | No | `deepseek-v4-flash` | Chat model name |
| `DEEPSEEK_BASE_URL` | No | `https://api.deepseek.com` | API base URL |
| `EMBEDDING_MODEL` | No | `all-MiniLM-L6-v2` | Sentence Transformers model used for embeddings |

## Limitations

- Scanned PDFs are detected but not processed with OCR. Users must provide searchable PDFs.
- PDF extraction quality depends on the embedded text and document layout.
- Chunking is character-based and page-aware; it is not agreement-specific or structurally aware of legal clauses.
- The vector store is ephemeral, so indexed vectors are not intended to persist across sessions.
- Generated answers depend on retrieved evidence and the external language-model API. Citations help trace evidence but do not guarantee that an answer is legally correct.
- This application is a document exploration tool, not a substitute for professional legal advice.
- A DeepSeek API key and network access are needed for generated answers; direct vector search and document exploration do not require a generation call.

## Future Improvements

- Add OCR support for scanned PDFs.
- Evaluate retrieval quality using a question-to-source-page test set.
- Experiment with structure-aware chunking and optional reranking.
- Add stronger citation validation and answer-quality checks.
- Add tests for adversarial instructions embedded in uploaded documents.
- Provide optional persistence with explicit document retention and deletion controls.

## Author

**Madhav Kumar**
