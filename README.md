# GraphLLM - PDF Knowledge Graph & RAG System

A comprehensive system for extracting knowledge graphs from PDFs and providing intelligent question-answering with citations using Retrieval-Augmented Generation (RAG).

## Features

- **PDF Processing**: Extract text, tables, code blocks, and images from PDFs
- **Knowledge Graph**: Automatically build semantic knowledge graphs with entity canonicalization
- **Vector Search**: FAISS-powered semantic search for efficient retrieval
- **RAG Chat**: Ask questions with page-cited answers, using Agentic AI for answering
- **Node Summarization**: Click any graph node for AI-generated summaries
- **Dark UI**: Sleek, accessible HTML/CSS interface

## Architecture

### Components

1. **PDF Ingestion** (`pdf_processor.py`)
   - PyMuPDF for text extraction
   - pdfplumber for tables
   - Tesseract OCR for images
   - Heuristic code block detection

2. **Embeddings** (`embedding_service.py`)
   - SentenceTransformers (multi-qa-MiniLM-L6-cos-v1)
   - FAISS vector index for fast retrieval

3. **Triplet Extraction** (`triplet_extractor.py`)
   - Hybrid approach: spaCy + LLM canonicalization
   - Entity similarity-based deduplication

4. **Knowledge Graph** (`graph_store.py`, `graph_builder.py`)
   - NetworkX (local) or Neo4j (production)
   - PageRank-based importance scoring
   - Configurable pruning

5. **LLM Layer** (`llm_service.py`)
   - Mistral 7B for generation
   - Structured prompts for extraction, summarization, chat

6. **FastAPI Backend** (`main.py`)
   - `/upload` - PDF upload and processing
   - `/graph` - Get knowledge graph
   - `/node/{id}` - Node details with summary
   - `/chat` - RAG chat with citations
   - `/admin/*` - Admin endpoints

7. **Frontend** (`frontend/`)
   - Dark-themed HTML/CSS
   - Graph visualization container
   - Chat interface
   - Node detail pane

## Installation

### Prerequisites

- Python 3.10+
- Tesseract OCR
- PostgreSQL (optional, for metadata)
- Neo4j (optional, for graph storage)

### Quick Start

1. **Clone and setup**
   ```bash
   cd GraphLLM
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

2. **Download spaCy model**
   ```bash
   python -m spacy download en_core_web_sm
   ```

3. **Configure environment**
   ```bash
   cp .env.example .env
   # Edit .env and add your MISTRAL_API_KEY
   ```

4. **Run the application**
   ```bash
   python main.py
   ```

5. **Open browser**
   Navigate to `http://localhost:8000`

### Docker Deployment

1. **Build and run**
   ```bash
   docker-compose up -d
   ```

2. **Access services**
   - Application: http://localhost:8000
   - Neo4j Browser: http://localhost:7474
   - Grafana: http://localhost:3000

## Configuration

Key settings in `.env`:

```bash
# LLM Settings
MISTRAL_API_KEY=your_api_key
MISTRAL_MODEL=mistral-7b-instruct-v0.1
LLM_TEMPERATURE=0.3

# Embedding
EMBEDDING_MODEL=sentence-transformers/multi-qa-MiniLM-L6-cos-v1

# Chunking
CHUNK_SIZE=512
CHUNK_OVERLAP=128

# Graph Pruning
NODE_IMPORTANCE_THRESHOLD=0.3
EDGE_CONFIDENCE_THRESHOLD=0.5

# RAG
RAG_TOP_K=10
```

## Usage

### Upload a PDF

1. Click "Upload PDF" in the header
2. Select a PDF file (max 50MB)
3. Wait for processing (progress shown in status)

### Explore the Graph

- Nodes appear in the graph visualization pane
- Click a node to view:
  - AI-generated summary
  - Supporting sources with page numbers
  - Related nodes

### Chat with Your Document

1. Type a question in the chat input
2. Press Enter or click "Send"
3. Receive an answer with inline citations (p. N)
4. View source snippets below the answer

### Admin Functions

- **Reindex**: Re-process a PDF
- **Clear All**: Delete all data (use with caution)

## API Reference

### Endpoints

#### `POST /upload`
Upload a PDF for processing.

**Request**: `multipart/form-data` with file

**Response**:
```json
{
  "pdf_id": "uuid",
  "filename": "example.pdf",
  "status": "processing",
  "message": "PDF uploaded successfully"
}
```

#### `GET /graph?pdf_id={id}`
Get knowledge graph nodes and edges.

**Response**:
```json
{
  "nodes": [...],
  "edges": [...],
  "metadata": {"total_nodes": 123, "total_edges": 456}
}
```

#### `GET /node/{node_id}`
Get detailed information about a node.

**Response**:
```json
{
  "node_id": "uuid",
  "label": "Entity Name",
  "type": "concept",
  "summary": "AI-generated summary with (p. 12) citations",
  "sources": [...],
  "related_nodes": [...]
}
```

#### `POST /chat`
Ask a question using RAG.

**Request**:
```json
{
  "query": "What is the main concept?",
  "pdf_id": "uuid",
  "include_citations": true,
  "max_sources": 5
}
```

**Response**:
```json
{
  "answer": "The main concept is... (p. 5)",
  "sources": [...]
}
```

#### `GET /admin/status`
Get system statistics.

## Prompt Templates

The system uses carefully crafted prompts for:

1. **Triplet Canonicalization**: Cleans raw triples into canonical form
2. **Node Summarization**: Generates summaries with citations
3. **RAG Chat**: Answers questions strictly from context

See `llm_service.py` for full templates.

## Graph Algorithms

### Entity Canonicalization
- Computes embeddings for all entities
- Merges entities with cosine similarity > 0.85
- Preserves aliases for merged entities

### Importance Scoring
Weighted combination of:
- Number of supporting chunks (30%)
- PageRank centrality (50%)
- Number of connections (20%)

### Pruning
- Removes nodes with importance < threshold
- Removes edges with confidence < threshold
- Preserves code/table entities

## Development

### Project Structure

```
GraphLLM/
├── main.py                 # FastAPI application
├── config.py              # Configuration management
├── models.py              # Pydantic data models
├── pdf_processor.py       # PDF extraction
├── embedding_service.py   # Vector search
├── llm_service.py         # LLM inference
├── triplet_extractor.py   # Knowledge extraction
├── graph_store.py         # Graph persistence
├── graph_builder.py       # Graph construction
├── frontend/              # HTML/CSS/JS
│   ├── index.html
│   ├── styles.css
│   └── app.js
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### Running Tests

```bash
pytest tests/
```

### Adding New Extractors

Extend `TripletExtractor` class:

```python
def _extract_custom_pattern(self, chunk: Chunk) -> List[Triple]:
    # Your extraction logic
    pass
```

## Scaling

### Production Recommendations

1. **Use Neo4j** for graph storage (set `use_neo4j=True`)
2. **Use GPU FAISS** for faster retrieval (`faiss-gpu`)
3. **Enable PostgreSQL** for metadata persistence
4. **Deploy with Kubernetes** for horizontal scaling
5. **Add Redis** for caching LLM responses
6. **Use LLM hosting** (e.g., Together.ai, Replicate) for better throughput

### Performance Tuning

- Adjust `CHUNK_SIZE` and `CHUNK_OVERLAP` for domain
- Tune `RAG_TOP_K` based on context window
- Increase `LLM_TEMPERATURE` for creative summaries
- Lower thresholds for denser graphs

## Monitoring

Prometheus metrics exposed on `/metrics` (if enabled):
- PDF processing time
- LLM latency
- Vector search performance
- Graph statistics

## Troubleshooting

### "Mistral API key not configured"
Set `MISTRAL_API_KEY` in `.env`

### "spaCy model not found"
Run `python -m spacy download en_core_web_sm`

### "Neo4j connection failed"
Check Neo4j is running and credentials are correct

### Graph visualization not working
Ensure JavaScript is enabled; integrate D3.js or vis.js for full visualization

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Submit a pull request

## License

MIT License - see LICENSE file

## Acknowledgments

- Built following comprehensive manual specification
- Uses Mistral 7B for generation
- FAISS for vector search
- Neo4j for graph storage
- FastAPI for backend
- spaCy for NLP

## Support

For issues and feature requests, please create an issue on GitHub.

---

**GraphLLM v1.0** - Transform PDFs into knowledge with AI
