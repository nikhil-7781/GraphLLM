# GraphLLM - Technical Deep Dive & Interview Prep

## Table of Contents
1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Data Flow](#data-flow)
4. [Technology Choices & Justifications](#technology-choices)
5. [Core Components](#core-components)
6. [Performance Optimizations](#performance-optimizations)
7. [Interview Q&A](#interview-qa)

---

## System Overview

**GraphLLM** is a production-grade **GraphRAG system** that combines:
- **Knowledge Graph extraction** from PDF documents
- **Vector-based semantic search** (FAISS embeddings)
- **Agent-based RAG** with multi-tool reasoning (LangGraph)
- **Hybrid retrieval** combining graph structure + vector similarity

### What Problem Does It Solve?

**Traditional RAG limitations:**
- Only retrieves similar text chunks (no relationship understanding)
- Misses connections between concepts
- Cannot do multi-hop reasoning
- No structured knowledge representation

**GraphRAG solution:**
- Extracts entities and relationships → Knowledge Graph
- Understands concept connections (A → B → C)
- Enables multi-hop reasoning ("How does X relate to Y?")
- Combines structured (graph) + unstructured (text) retrieval

---

## Architecture

### High-Level System Design

```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Upload  │  │  Graph   │  │   Node   │  │   Chat   │   │
│  │   PDF    │  │   Viz    │  │  Details │  │    UI    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        │             ▼             ▼             ▼
┌───────┼─────────────────────────────────────────────────────┐
│       │              FastAPI Backend                        │
│       ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Ingestion Pipeline                      │   │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  │   │
│  │  │ PDF  │→ │Chunk │→ │Embed │→ │Extract│→ │Build │  │   │
│  │  │Parse │  │ Text │  │ding  │  │Triple │  │Graph │  │   │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Query Pipeline (Agent-Based)            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│  │  │  Plan    │→ │ Execute  │→ │Synthesize│→ Answer  │   │
│  │  │  Tools   │  │  Tools   │  │ Response │          │   │
│  │  └──────────┘  └──────────┘  └──────────┘          │   │
│  │                                                      │   │
│  │  Available Tools:                                   │   │
│  │  • vector_search   (semantic similarity)           │   │
│  │  • graph_search    (find concepts)                 │   │
│  │  • get_node_details (node info)                    │   │
│  │  • get_related_nodes (traverse graph)              │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│    FAISS     │  │  NetworkX    │  │   Gemini     │
│   Vector     │  │  Knowledge   │  │     API      │
│   Index      │  │    Graph     │  │   (LLM)      │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Architecture Pattern: **Modular Microservices-Style**

Each component is a separate service/class with clear responsibilities:

1. **PDFProcessor** - PDF parsing & text extraction
2. **EmbeddingService** - Vector embeddings + FAISS index
3. **GeminiExtractor** - Entity/relationship extraction
4. **GraphBuilder** - Knowledge graph construction
5. **GraphStore** - Graph storage & queries
6. **LLMService** - All LLM interactions
7. **RAGAgent** - Agent-based query orchestration

**Why this pattern?**
- ✅ **Testable** - Each component can be tested independently
- ✅ **Replaceable** - Swap FAISS → Pinecone, NetworkX → Neo4j
- ✅ **Scalable** - Can distribute services across machines
- ✅ **Maintainable** - Clear separation of concerns

---

## Data Flow

### 1. PDF Upload & Ingestion (30-60 seconds)

```
User uploads PDF
      ↓
┌─────────────────────────────────────────────────────────┐
│ 1. PDF Parsing (pdf_processor.py)                      │
│    - Extract text with PyMuPDF                         │
│    - OCR images with Tesseract (if needed)             │
│    - Extract tables with Camelot                       │
│    - Output: List[Chunk] with page numbers            │
│    Time: 2-5s for 10-page PDF                         │
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Text Chunking (pdf_processor.py)                    │
│    - Chunk size: 512 tokens                            │
│    - Overlap: 128 tokens                               │
│    - Preserve sentence boundaries                      │
│    - Output: ~20-30 chunks per 10 pages                │
│    Time: < 1s                                          │
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│ 3. Embedding Generation (embedding_service.py)         │
│    - Model: sentence-transformers/multi-qa-MiniLM-L6   │
│    - Batch size: 128 (optimized)                       │
│    - Dimension: 384                                    │
│    - Build FAISS HNSW index                            │
│    Time: 3-5s for 30 chunks                            │
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│ 4. Parallel Triplet Extraction (gemini_extractor.py)   │
│    - Process all pages in parallel (asyncio.gather)    │
│    - Gemini extracts 2 concepts per page (max)         │
│    - Temperature=0.0 (deterministic)                   │
│    - Output: Subject-Predicate-Object triples          │
│    Example: (Graph Neural Networks, uses, Message Passing)│
│    Time: 5-8s for 10 pages (parallel)                  │
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│ 5. Graph Construction (graph_builder.py)               │
│    - Create nodes from entities                        │
│    - Create edges from relationships                   │
│    - Compute importance scores (degree centrality)     │
│    - Store in NetworkX MultiGraph (undirected)         │
│    - Cache graph + embeddings to disk                  │
│    Time: 1-2s                                          │
└─────────────────────────────────────────────────────────┘
      ↓
Graph ready for queries!
```

**Total Time:** 10-20 seconds for 10-page PDF (5-6x faster than sequential)

---

### 2. Query Processing (Agent-Based, 1-3 seconds)

```
User asks: "How do Graph Neural Networks work?"
      ↓
┌─────────────────────────────────────────────────────────┐
│ PLAN Node (rag_agent.py)                               │
│ Agent analyzes query and decides which tools to use:   │
│   • "explain" keyword → use graph_search               │
│   • Technical concept → use vector_search              │
│   • Planning: [vector_search, graph_search]            │
│ Time: < 10ms                                           │
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│ EXECUTE TOOLS Node (rag_agent.py)                      │
│                                                         │
│ Tool 1: vector_search("Graph Neural Networks")         │
│   → FAISS finds top 5 similar chunks                   │
│   → Returns: [chunk1, chunk2, chunk3, chunk4, chunk5]  │
│   Time: 50-100ms                                       │
│                                                         │
│ Tool 2: graph_search("Graph Neural Networks")          │
│   → NetworkX finds node by label                       │
│   → Returns: {node_id, label, type, importance}        │
│   Time: 5-10ms                                         │
│                                                         │
│ Tool 3: get_related_nodes(node_id)                     │
│   → NetworkX traverses edges                           │
│   → Returns: [(neighbor1, relation), (neighbor2, ...)] │
│   Time: 5-10ms                                         │
│                                                         │
│ Total Tool Execution: 100-200ms                        │
└─────────────────────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────────────────────┐
│ SYNTHESIZE Node (rag_agent.py)                         │
│ Combine all tool results:                              │
│   • Vector search chunks (semantic context)            │
│   • Graph node info (concept definition)               │
│   • Related concepts (relationships)                   │
│                                                         │
│ Gemini synthesizes final answer:                       │
│   Input: Combined context (500-1000 tokens)            │
│   Output: Answer with citations                        │
│   Time: 800-1500ms                                     │
└─────────────────────────────────────────────────────────┘
      ↓
Return answer with sources to user
```

**Total Time:** 1-2 seconds (first time), < 100ms (cached nodes)

---

## Technology Choices

### 1. Why Gemini API over OpenAI/Mistral?

| Factor | Gemini 2.5-Flash | OpenAI GPT-4 | Mistral 7B |
|--------|------------------|--------------|------------|
| **Cost** | ~$0.0001/request | ~$0.01/request | ~$0.0005/request |
| **Speed** | 800-1500ms | 2000-3000ms | 1000-2000ms |
| **Quality** | ⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐⭐ Best | ⭐⭐⭐ Good |
| **Context** | 1M tokens | 128K tokens | 32K tokens |
| **Free Tier** | ✅ Generous | ❌ None | ✅ Limited |

**Decision:** Gemini 2.5-Flash
- Best cost/performance ratio
- Excellent at structured output (JSON triples)
- Fast enough for real-time responses
- Free tier sufficient for development

---

### 2. Why FAISS over Pinecone/Weaviate/Qdrant?

| Factor | FAISS | Pinecone | Weaviate | Qdrant |
|--------|-------|----------|----------|--------|
| **Setup** | ⭐⭐⭐⭐⭐ pip install | ⭐⭐ Cloud signup | ⭐⭐⭐ Docker | ⭐⭐⭐ Docker |
| **Cost** | ✅ Free | 💰 $70+/month | ✅ Free (self-host) | ✅ Free (self-host) |
| **Speed** | ⭐⭐⭐⭐⭐ Fastest | ⭐⭐⭐⭐ Fast | ⭐⭐⭐⭐ Fast | ⭐⭐⭐⭐ Fast |
| **Scalability** | ⭐⭐⭐ 1M vectors | ⭐⭐⭐⭐⭐ Billions | ⭐⭐⭐⭐ Millions | ⭐⭐⭐⭐ Millions |
| **Deployment** | ⭐⭐⭐⭐⭐ Embedded | ⭐⭐ Cloud-only | ⭐⭐⭐ Self-host | ⭐⭐⭐ Self-host |

**Decision:** FAISS (Facebook AI Similarity Search)
- No external dependencies (embedded in app)
- Extremely fast (C++ implementation)
- Perfect for < 1M vectors
- Easy deployment (no separate service)
- HNSW algorithm for approximate nearest neighbor

**When to switch:**
- > 10M vectors → Pinecone (managed service)
- Need filtering/metadata → Weaviate/Qdrant
- Multi-tenancy → Pinecone/Qdrant

---

### 3. Why NetworkX over Neo4j?

| Factor | NetworkX | Neo4j |
|--------|----------|-------|
| **Setup** | ⭐⭐⭐⭐⭐ pip install | ⭐⭐ Docker/Cloud |
| **Query Speed** | ⭐⭐⭐⭐ Fast (< 100K nodes) | ⭐⭐⭐⭐⭐ Fast (billions) |
| **Graph Algorithms** | ⭐⭐⭐⭐⭐ Rich library | ⭐⭐⭐⭐ Good (with plugins) |
| **Persistence** | ⭐⭐⭐ Pickle files | ⭐⭐⭐⭐⭐ ACID database |
| **Scalability** | ⭐⭐⭐ < 1M nodes | ⭐⭐⭐⭐⭐ Billions of nodes |
| **Deployment** | ⭐⭐⭐⭐⭐ Embedded | ⭐⭐ Separate service |

**Decision:** NetworkX (in-memory Python graph)
- Simple deployment (no database server)
- Perfect for < 100K nodes (PDF → ~100-1000 nodes)
- Fast for our use case (< 10ms queries)
- Easy to persist (pickle to disk)
- Rich algorithm library

**When to switch:**
- > 100K nodes → Neo4j
- Need ACID transactions → Neo4j
- Multi-user concurrent writes → Neo4j
- Complex graph queries (Cypher) → Neo4j

**Code supports both!** (`GraphStore` has `use_neo4j=False` flag)

---

### 4. Why LangGraph over LangChain/LlamaIndex?

| Factor | LangGraph | LangChain | LlamaIndex |
|--------|-----------|-----------|------------|
| **Agent Control** | ⭐⭐⭐⭐⭐ Full control | ⭐⭐⭐ Limited | ⭐⭐⭐ Limited |
| **State Management** | ⭐⭐⭐⭐⭐ Built-in | ⭐⭐ Manual | ⭐⭐ Manual |
| **Multi-hop** | ⭐⭐⭐⭐⭐ Native | ⭐⭐⭐ Chains | ⭐⭐⭐ Chains |
| **Tool Orchestration** | ⭐⭐⭐⭐⭐ Graph-based | ⭐⭐⭐ Sequential | ⭐⭐⭐ Sequential |
| **Debugging** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐ Good | ⭐⭐⭐ Good |

**Decision:** LangGraph (from LangChain team)
- **Explicit control flow** with StateGraph
- **Stateful workflows** (plan → execute → synthesize)
- **Conditional branching** (use tools vs direct answer)
- **Built-in agent patterns** for tool use
- **Debuggable** - can inspect state at each step

**Agent Workflow:**
```python
workflow = StateGraph(AgentState)
workflow.add_node("plan", plan_node)        # Decide which tools
workflow.add_node("execute", execute_node)   # Run tools
workflow.add_node("synthesize", synth_node)  # Combine results
workflow.add_conditional_edges(...)          # Dynamic routing
```

---

### 5. Why Sentence-Transformers over OpenAI Embeddings?

| Factor | Sentence-Transformers | OpenAI Embeddings |
|--------|----------------------|-------------------|
| **Cost** | ✅ Free | 💰 $0.0001/1K tokens |
| **Speed** | ⭐⭐⭐⭐⭐ 10-50ms | ⭐⭐⭐ 200-500ms |
| **Offline** | ✅ Yes | ❌ API-only |
| **Quality** | ⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐⭐ Best |
| **Dimension** | 384 (configurable) | 1536 (fixed) |

**Decision:** Sentence-Transformers (multi-qa-MiniLM-L6)
- **Free** and **fast** (local inference)
- **Works offline** (no API calls)
- **Good enough** for semantic search
- **Small model** (~90MB) - easy to deploy

**Model Choice: multi-qa-MiniLM-L6-cos-v1**
- Optimized for Q&A retrieval
- 384 dimensions (4x smaller than OpenAI)
- Fast inference (10-20ms per batch)
- Good semantic understanding

---

## Core Components

### 1. PDFProcessor (pdf_processor.py)

**Responsibility:** Extract text from PDF documents

**Key Features:**
- Multi-format support (native text, scanned images, tables)
- OCR with Tesseract for images
- Table extraction with Camelot
- Smart chunking with overlap

**Code Highlights:**
```python
def process_pdf(self, filepath: str, pdf_id: str) -> Tuple[List[Chunk], Dict]:
    # 1. Extract text with PyMuPDF (fast)
    # 2. Detect scanned pages → OCR with Tesseract
    # 3. Extract tables with Camelot
    # 4. Return chunks with metadata (page numbers, text, etc.)
```

**Why PyMuPDF?**
- 10x faster than PyPDF2
- Better text extraction quality
- Handles complex PDFs (embedded fonts, images)

---

### 2. EmbeddingService (embedding_service.py)

**Responsibility:** Vector embeddings + similarity search

**Key Features:**
- Sentence-Transformers for embedding generation
- FAISS HNSW index for fast approximate search
- Batch processing (128 chunks at once)
- Persistent storage (save/load index)

**Code Highlights:**
```python
class EmbeddingService:
    def create_embeddings(self, chunks: List[Chunk]) -> List[EmbeddingEntry]:
        # Batch encode with sentence-transformers
        embeddings = self.model.encode(
            texts,
            batch_size=128,  # Optimized for speed
            normalize_embeddings=True
        )
        return embeddings
    
    def search(self, query: str, top_k: int = 5):
        # 1. Embed query
        # 2. FAISS search (HNSW approximate)
        # 3. Return top-k with scores
        scores, indices = self.index.search(query_vector, top_k)
```

**FAISS Index Type: IndexHNSWFlat**
- **HNSW** = Hierarchical Navigable Small World
- **Approximate** nearest neighbor (99% accuracy, 10x faster)
- **Trade-off:** Speed vs accuracy (we choose speed)

---

### 3. GeminiExtractor (gemini_extractor.py)

**Responsibility:** Extract knowledge graph triples from text

**Key Features:**
- Parallel page processing (asyncio.gather)
- Structured JSON output (subject-predicate-object)
- 25 canonical relation types
- Generic concept filtering (avoid "data", "system", etc.)
- Deterministic extraction (temperature=0)

**Code Highlights:**
```python
async def extract_from_chunks(self, chunks: List[Chunk]):
    # Group chunks by page
    chunks_by_page = group_by_page(chunks)
    
    # ⚡ PARALLEL: Process all pages simultaneously
    tasks = [
        self._extract_with_gemini(page_text, page_num)
        for page_num, page_text in chunks_by_page.items()
    ]
    results = await asyncio.gather(*tasks)
    
    # Results: List[CanonicalTriple]
    # Example: (Graph Neural Networks, uses, Message Passing)
```

**Prompt Engineering:**
```
Extract 2 most important technical concepts from this page.
For each concept, identify relationships to other concepts.

Output JSON:
{
  "nodes": [
    {"node1": "Graph Neural Networks", "node2": "Message Passing", "relation": "uses"},
    ...
  ]
}

CRITICAL: Use ONLY these 25 canonical relations:
- is_a, part_of, uses, causes, defined_as, related_to, ...
```

**Why 25 canonical relations?**
- Prevents relation explosion ("utilizes" vs "uses" vs "applies")
- Easier to query ("show all X that *uses* Y")
- Better graph consistency

---

### 4. GraphBuilder (graph_builder.py)

**Responsibility:** Build knowledge graph from extracted triples

**Key Features:**
- Entity canonicalization (merge similar entities) - SKIPPED for speed
- Node creation with supporting chunks
- Edge creation with confidence scores
- Importance scoring (degree centrality)
- Graph pruning (remove low-importance nodes) - SKIPPED for speed

**Code Highlights:**
```python
async def build_graph(self, triples: List[CanonicalTriple]):
    # 1. Canonicalize entities (SKIPPED - identity mapping)
    entity_map = {entity: entity for entity in unique_entities}
    
    # 2. Create nodes
    for entity in entity_map:
        node = create_node(entity, supporting_chunks)
        graph_store.add_node(node)
    
    # 3. Create edges
    for triple in triples:
        edge = create_edge(triple)
        graph_store.add_edge(edge)
    
    # 4. Compute importance (degree centrality)
    for node in graph.nodes:
        node.importance = num_neighbors / 10.0
    
    # 5. Pruning SKIPPED (all nodes kept)
```

**Why skip canonicalization?**
- Gemini already extracts clean, specific entities
- Embedding-based similarity too slow (O(n²))
- 2-concept limit per page → minimal duplicates
- 5-6x speedup

**Why skip pruning?**
- Better to keep all extracted concepts
- User can explore full graph
- Filtering can be done at query time

---

### 5. GraphStore (graph_store.py)

**Responsibility:** Store and query knowledge graph

**Key Features:**
- NetworkX MultiGraph (undirected)
- In-memory storage (fast)
- Persistent storage (pickle)
- Query operations (get neighbors, find nodes by label)
- Neo4j support (disabled, but code ready)

**Code Highlights:**
```python
class GraphStore:
    def __init__(self, use_neo4j=False):
        self.graph = nx.MultiGraph()  # Undirected graph
        self.nodes_dict = {}  # node_id → GraphNode
        self.edges_dict = {}  # edge_id → GraphEdge
    
    def get_neighbors(self, node_id: str):
        # NetworkX undirected graph traversal
        for neighbor_id in self.graph.neighbors(node_id):
            neighbor_node = self.nodes_dict[neighbor_id]
            edges = self.graph.get_edge_data(node_id, neighbor_id)
            yield (neighbor_node, edge)
```

**Why MultiGraph?**
- Allows multiple edges between same nodes
- Example: (A, uses, B) and (A, enhances, B) coexist

**Why undirected?**
- Relationships are bidirectional (A uses B ⟷ B used-by A)
- Simpler traversal (no need to check both directions)
- Cleaner visualization (no arrows)

---

### 6. LLMService (llm_service.py)

**Responsibility:** All LLM interactions

**Key Features:**
- Centralized prompt templates
- Gemini API integration via litellm
- Retry logic (3 attempts with exponential backoff)
- JSON mode for structured output
- Multiple use cases (canonicalization, summarization, chat, agent synthesis)

**Code Highlights:**
```python
class LLMService:
    async def _call_api(self, messages, temperature=0.7, json_mode=False):
        # litellm wrapper for Gemini
        response = await asyncio.to_thread(
            self.litellm.completion,
            model="gemini/gemini-2.5-flash",
            api_key=self.api_key,
            messages=messages,
            temperature=temperature,
            max_tokens=max_tokens,
            response_format={"type": "json_object"} if json_mode else None
        )
        return response.choices[0].message.content
    
    async def summarize_node(self, node_label, chunks):
        # Generate concise summary with citations
        prompt = f"Summarize {node_label} using these chunks..."
        return await self._call_api([{"role": "user", "content": prompt}])
```

**Why litellm?**
- Unified API for multiple LLM providers (Gemini, OpenAI, etc.)
- Easy to switch models
- Built-in retry and error handling

---

### 7. RAGAgent (rag_agent.py) - **THE CORE INNOVATION**

**Responsibility:** Intelligent query orchestration with multi-tool reasoning

**Key Features:**
- LangGraph StateGraph workflow
- 5 tool functions (vector_search, graph_search, get_node_details, get_related_nodes, get_chunk_by_id)
- Plan → Execute → Synthesize pattern
- Fallback to simple RAG if agent fails

**Agent Architecture:**
```
┌─────────────────────────────────────────────────┐
│                  AgentState                     │
│  • messages: conversation history               │
│  • query: user question                         │
│  • tool_results: dict of tool outputs           │
│  • reasoning_steps: agent's thinking            │
│  • final_answer: synthesized answer             │
│  • citations: source references                 │
└─────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│         PLAN NODE (plan_node)                   │
│  Analyzes query, decides which tools to use:    │
│  • Keywords: "relate" → graph_search            │
│  • Keywords: "explain" → graph_search           │
│  • Always: vector_search (semantic baseline)    │
│  Output: list of tools to execute               │
└─────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│      EXECUTE TOOLS NODE (execute_tools_node)    │
│  Runs planned tools in sequence:                │
│  1. vector_search(query, top_k=5)               │
│     → Returns: similar text chunks              │
│  2. graph_search(concept)                       │
│     → Returns: graph node info                  │
│  3. get_related_nodes(node_id)                  │
│     → Returns: neighboring concepts             │
│  All results stored in state.tool_results       │
└─────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────┐
│      SYNTHESIZE NODE (synthesize_node)          │
│  Combines tool results:                         │
│  • Vector chunks → semantic context             │
│  • Graph nodes → concept definitions            │
│  • Related nodes → relationship context         │
│  Gemini generates final answer with citations   │
└─────────────────────────────────────────────────┘
               │
               ▼
          Final Answer
```

**Why Agent-Based?**
- **Traditional RAG:** Just vector search
- **Agent RAG:** Dynamically chooses which retrieval strategy
- **Multi-hop:** Can follow graph edges (A → B → C)
- **Richer context:** Combines multiple information sources

**Code Highlights:**
```python
def _plan_node(self, state: AgentState) -> AgentState:
    query = state["query"]
    tools_to_use = ["vector_search"]  # Always start with semantic
    
    if "relate" in query.lower():
        tools_to_use.append("graph_search")
    if "what is" in query.lower():
        tools_to_use.append("graph_search")
    
    state["tool_results"] = {"planned_tools": tools_to_use}
    return state

def _execute_tools_node(self, state: AgentState):
    for tool_name in state["tool_results"]["planned_tools"]:
        result = execute_tool(tool_name, state["query"])
        state["tool_results"][tool_name] = result
    return state

async def _synthesize_node(self, state: AgentState):
    # Combine all tool results into context
    context = format_tool_results(state["tool_results"])
    answer = await llm_service.agent_synthesize(query, context)
    state["final_answer"] = answer
    return state
```

**5 Tools Explained:**

1. **vector_search** - Semantic similarity search
   ```python
   def vector_search(query: str, top_k=5) -> List[Dict]:
       # Find chunks similar to query using FAISS
       results = embedding_service.search(query, top_k)
       return [{"text": chunk, "page": page, "score": score}]
   ```

2. **graph_search** - Find concept in knowledge graph
   ```python
   def graph_search(concept: str) -> Dict:
       # Find node by label (case-insensitive)
       node = graph_store.get_node_by_label(concept)
       return {"node_id": id, "label": label, "importance": score}
   ```

3. **get_node_details** - Get full node information
   ```python
   def get_node_details(node_id: str) -> Dict:
       node = graph_store.get_node(node_id)
       return {
           "label": node.label,
           "type": node.type,
           "supporting_chunks": node.supporting_chunks[:5]
       }
   ```

4. **get_related_nodes** - Graph traversal
   ```python
   def get_related_nodes(node_id: str, max=5) -> List[Dict]:
       neighbors = graph_store.get_neighbors(node_id)
       return [
           {"label": n.label, "relation": edge.relation}
           for n, edge in neighbors[:max]
       ]
   ```

5. **get_chunk_by_id** - Retrieve specific chunk
   ```python
   def get_chunk_by_id(chunk_id: str) -> Dict:
       # Direct chunk retrieval for detailed citations
       chunk = embedding_service.get_chunk(chunk_id)
       return {"text": chunk.text, "page": chunk.page}
   ```

---

## Performance Optimizations

### 1. **Parallel Page Processing** (5-6x speedup)

**Before:**
```python
for page in pages:
    triples = await extract_triples(page)  # Sequential
# Total: 60s for 10 pages (6s per page)
```

**After:**
```python
tasks = [extract_triples(page) for page in pages]
results = await asyncio.gather(*tasks)  # Parallel
# Total: 8s for 10 pages (all at once)
```

---

### 2. **Skip Canonicalization** (2x speedup)

**Problem:** Merging similar entities with embeddings is O(n²)

**Solution:** Use identity mapping (each entity maps to itself)
```python
# OLD: Compute similarity for all entity pairs → 30s
entity_map = compute_similarities(entities)

# NEW: Identity mapping → instant
entity_map = {entity: entity for entity in entities}
```

**Trade-off:**
- ❌ Might have duplicate entities ("GNN" vs "Graph Neural Networks")
- ✅ Much faster, Gemini already extracts clean entities

---

### 3. **Skip Graph Pruning** (instant)

**Problem:** PageRank computation is expensive (O(n × k) iterations)

**Solution:** Keep all extracted concepts, filter at query time
```python
# OLD: Compute PageRank, remove low-importance nodes → 10s
prune_graph(importance_threshold=0.3)

# NEW: Keep everything → 0s
# Users can explore full graph
```

---

### 4. **PDF Chunk Caching** (2-3s saved on reindex)

**Problem:** Re-extracting PDF text on every reindex is slow

**Solution:** Cache extracted chunks to pickle file
```python
cache_path = f"data/chunks_{pdf_id}.pkl"
if os.exists(cache_path):
    chunks = pickle.load(cache_path)  # 100ms
else:
    chunks = extract_pdf(file)  # 3000ms
    pickle.dump(chunks, cache_path)
```

---

### 5. **Increased Embedding Batch Size** (1.5x speedup)

**Problem:** Small batches → more GPU overhead

**Solution:** Increase from 32 → 128
```python
embeddings = model.encode(texts, batch_size=128)  # 3s vs 5s
```

---

### 6. **Node Summary Caching** (100x speedup for repeat clicks)

**Problem:** Every node click calls Gemini API (1-2s)

**Solution:** Cache summaries in node metadata
```python
if "cached_summary" in node.metadata:
    return node.metadata["cached_summary"]  # < 1ms
else:
    summary = await llm.summarize(node)  # 1500ms
    node.metadata["cached_summary"] = summary
    graph_store.update_node(node)
```

---

### 7. **Deterministic Extraction** (temperature=0)

**Problem:** Different graphs on every run (non-deterministic)

**Solution:** Set temperature=0 for consistent results
```python
response = await gemini.completion(
    prompt,
    temperature=0.0  # Deterministic (same input → same output)
)
```

Also sort entity lists for consistent ordering.

---

## Interview Q&A

### Architecture & Design

**Q: Why did you choose GraphRAG over traditional RAG?**

**A:** Traditional RAG has limitations:
- Only retrieves similar text (no relationship understanding)
- Misses multi-hop reasoning ("How does X relate to Y through Z?")
- No structured knowledge representation

GraphRAG solves this by:
1. Extracting entities + relationships → Knowledge Graph
2. Enabling graph traversal (multi-hop queries)
3. Combining vector search (semantic) + graph search (structural)

Example: "How do transformers relate to attention?"
- Traditional RAG: Finds chunks mentioning both (might miss connection)
- GraphRAG: Follows graph edges (Transformers → uses → Attention Mechanism)

---

**Q: Why use NetworkX instead of Neo4j?**

**A:** Trade-offs:

**NetworkX (our choice):**
- ✅ Simple deployment (no database server)
- ✅ Fast for < 100K nodes (our PDFs → 100-1000 nodes)
- ✅ Rich Python library (PageRank, centrality, etc.)
- ✅ Easy persistence (pickle files)
- ❌ Not scalable to millions of nodes
- ❌ No concurrent writes

**Neo4j:**
- ✅ Scales to billions of nodes
- ✅ ACID transactions, concurrent writes
- ✅ Advanced queries (Cypher language)
- ❌ Requires separate database service
- ❌ More complex deployment

**Decision:** NetworkX is perfect for our use case (single-user, < 100K nodes). The code supports both via `use_neo4j` flag for future migration.

---

**Q: How does the agent decide which tools to use?**

**A:** The agent uses **heuristic-based planning** (can be upgraded to LLM-based):

```python
def _plan_node(self, state):
    query = state["query"].lower()
    tools = ["vector_search"]  # Always use semantic search
    
    # Keyword-based tool selection
    if any(word in query for word in ["relate", "connection", "link"]):
        tools.append("graph_search")
        tools.append("get_related_nodes")
    
    if any(word in query for word in ["what is", "define", "explain"]):
        tools.append("graph_search")
    
    return tools
```

**Future improvement:** Use LLM to plan
```python
plan_prompt = f"Query: {query}\nAvailable tools: {tools}\nWhich tools should I use?"
planned_tools = await llm.plan(plan_prompt)
```

Trade-off: Heuristics are faster (< 1ms) vs LLM planning is smarter but slower (~500ms).

---

### Technical Deep-Dive

**Q: Walk me through the entire data flow from PDF upload to answering a query.**

**A:** 

**Ingestion (15-20s for 10-page PDF):**
1. **PDF Parsing** (3s): PyMuPDF extracts text + metadata
2. **Chunking** (< 1s): Split into 512-token chunks with 128-token overlap
3. **Embedding** (4s): sentence-transformers generates 384-dim vectors, FAISS builds HNSW index
4. **Extraction** (6s): Gemini extracts triples from all pages in parallel (asyncio.gather)
5. **Graph Building** (2s): Create NetworkX graph (nodes + edges), compute importance

**Query (1-2s):**
1. **Planning** (< 10ms): Agent decides which tools to use
2. **Tool Execution** (200ms):
   - vector_search: FAISS finds top 5 similar chunks (100ms)
   - graph_search: Find concept in graph (5ms)
   - get_related_nodes: Traverse edges (5ms)
3. **Synthesis** (1500ms): Gemini combines all results into final answer with citations

**Result:** Answer with inline citations returned to frontend

---

**Q: How do you handle PDF extraction quality issues (scanned images, tables)?**

**A:** Multi-strategy approach:

1. **Native text extraction** (PyMuPDF) - fastest, 90% of PDFs
2. **OCR for scanned pages** (Tesseract) - detect low text density → OCR image
3. **Table extraction** (Camelot) - preserves table structure as markdown
4. **Fallback** - If all fail, warn user about extraction quality

```python
if len(extracted_text) < 100:  # Likely scanned
    logger.warning(f"Page {page_num} has little text, trying OCR...")
    image = page.get_pixmap()
    text = pytesseract.image_to_string(image)
```

**Quality metrics:**
- Character count per page (detect scans)
- Text density (detect images vs text)
- Table detection (use Camelot for structured data)

---

**Q: How do you ensure consistent graph generation across runs?**

**A:** Three mechanisms:

1. **Deterministic LLM** (temperature=0):
   ```python
   response = gemini.completion(prompt, temperature=0.0)
   # Same input → same output
   ```

2. **Sorted entity processing**:
   ```python
   entities_list = sorted(list(entities))  # Alphabetical order
   ```

3. **Fixed processing order**:
   ```python
   for page_num in sorted(chunks_by_page.keys()):  # Pages in order
       extract_triples(page_num)
   ```

**Result:** Re-uploading the same PDF produces identical graphs (same nodes, same edges, same layout).

---

**Q: What happens if the Gemini API fails during extraction?**

**A:** Graceful degradation with retry logic:

1. **Retry with exponential backoff** (tenacity library):
   ```python
   @retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
   async def _call_api(self, messages):
       # Try up to 3 times with 1s, 2s, 4s delays
   ```

2. **Per-page error handling**:
   ```python
   results = await asyncio.gather(*tasks, return_exceptions=True)
   for page_num, result in zip(page_numbers, results):
       if isinstance(result, Exception):
           logger.error(f"Page {page_num} failed: {result}")
           continue  # Skip failed page, continue with others
   ```

3. **Partial graph construction**:
   - If 10 pages, 1 fails → still build graph with 9 pages
   - User gets partial but usable results

4. **Fallback for queries**:
   ```python
   try:
       answer = await rag_agent.chat(query)
   except Exception:
       logger.warning("Agent failed, falling back to simple RAG")
       answer = await simple_rag_fallback(query)
   ```

---

**Q: How do you handle disambiguation (e.g., "GNN" vs "Graph Neural Network")?**

**A:** Current approach (simplified):

**During extraction:**
- Rely on Gemini to extract full, unambiguous names
- Prompt explicitly asks for full technical terms
- Example: "Extract full concept names (e.g., 'Graph Neural Networks', not 'GNN')"

**Post-extraction (SKIPPED for speed):**
- Could use entity canonicalization with embeddings
- Compute similarity between all entity pairs
- Merge entities above threshold (e.g., 0.9 similarity)

**Trade-off:**
- ✅ Faster without canonicalization (2x speedup)
- ❌ Might have duplicates ("GNN" and "Graph Neural Networks" as separate nodes)
- ✅ Mitigated by good prompt engineering

**Future improvement:**
- Use entity linking to knowledge bases (Wikidata, DBpedia)
- Fuzzy matching for acronyms

---

### Performance & Scalability

**Q: How would you scale this to 1000 PDFs or 10,000 PDFs?**

**A:** Architecture changes by scale:

**Current (1-100 PDFs):**
- Single FastAPI instance
- In-memory FAISS + NetworkX
- Pickle persistence
- Works perfectly

**Medium scale (100-1000 PDFs):**
1. **Separate indexing service**
   - Background workers for PDF processing
   - Message queue (Celery + Redis) for async jobs
   
2. **Persistent storage**
   - PostgreSQL for metadata
   - S3/GCS for PDF storage
   - Keep FAISS + NetworkX (still manageable)

3. **Caching layer**
   - Redis for API response caching
   - CDN for frontend assets

**Large scale (1000-10K PDFs):**
1. **Distributed vector search**
   - Replace FAISS → Pinecone/Weaviate (managed service)
   - Or Qdrant cluster (self-hosted, horizontally scalable)

2. **Graph database**
   - Replace NetworkX → Neo4j cluster
   - Enables multi-user concurrent access
   - Advanced graph queries with Cypher

3. **Microservices**
   - Separate services: Ingestion, Query, Graph, Vector
   - Kubernetes orchestration
   - Load balancer + auto-scaling

4. **Distributed processing**
   - Apache Spark for parallel PDF processing
   - Ray for distributed embedding generation

**Architecture:**
```
Load Balancer
     ↓
┌────────────────┐
│  API Gateway   │
└────────────────┘
     ↓
┌────────────────────────────────────┐
│  Microservices                     │
│  ┌──────┐  ┌──────┐  ┌──────┐     │
│  │Query │  │Ingest│  │Graph │     │
│  │Service  │Service  │Service     │
│  └──────┘  └──────┘  └──────┘     │
└────────────────────────────────────┘
     ↓              ↓           ↓
┌─────────┐  ┌──────────┐  ┌──────┐
│Pinecone │  │PostgreSQL│  │Neo4j │
│(Vector) │  │(Metadata)│  │(Graph)│
└─────────┘  └──────────┘  └──────┘
```

---

**Q: What are the current bottlenecks?**

**A:** Performance analysis:

**Ingestion bottlenecks:**
1. **Gemini API calls** (6s for 10 pages)
   - Already optimized with parallel processing
   - Further improvement: batch API requests (if supported)

2. **Embedding generation** (4s for 30 chunks)
   - Already using batch_size=128
   - Further improvement: GPU inference (10x faster)

3. **PDF extraction** (3s)
   - Cached after first extraction
   - Further improvement: parallel page extraction

**Query bottlenecks:**
1. **Gemini API synthesis** (1.5s)
   - Biggest single bottleneck
   - Solutions: Smaller model (Gemini-1.5-flash), shorter context, streaming response

2. **FAISS search** (100ms)
   - Acceptable, but could optimize
   - Solutions: Smaller index (fewer vectors), IVF index (faster approximate search)

3. **Graph traversal** (5-10ms)
   - Not a bottleneck
   - NetworkX is very fast for < 100K nodes

**Current bottleneck: Gemini API latency** (1.5s for synthesis)

---

**Q: How would you add multi-tenancy (multiple users)?**

**A:** Design changes:

**1. User isolation:**
```python
# PDF metadata
pdf_id = f"{user_id}_{uuid.uuid4()}"

# Vector index per user
faiss_index_path = f"data/users/{user_id}/faiss_index"

# Graph per user
graph_path = f"data/users/{user_id}/knowledge_graph.pkl"
```

**2. Database schema:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR,
    api_quota INT
);

CREATE TABLE pdfs (
    pdf_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users,
    filename VARCHAR,
    upload_date TIMESTAMP,
    status VARCHAR  -- processing, completed, failed
);

CREATE TABLE graphs (
    graph_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users,
    pdf_id UUID REFERENCES pdfs,
    num_nodes INT,
    num_edges INT
);
```

**3. API authentication:**
```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def get_current_user(token: str = Depends(security)):
    user = verify_token(token)  # JWT verification
    if not user:
        raise HTTPException(401, "Invalid token")
    return user

@app.post("/upload")
async def upload_pdf(file, user: User = Depends(get_current_user)):
    # Process PDF for this user only
    pdf_id = f"{user.user_id}_{uuid.uuid4()}"
    ...
```

**4. Shared vs isolated graphs:**

**Option A: Isolated (simple, secure)**
- Each user has separate FAISS index + graph
- No data leakage
- More storage (duplicate indices)

**Option B: Shared with filters (efficient)**
- One FAISS index, filter by user_id metadata
- One Neo4j database, filter queries by user_id
- Less storage
- Requires careful access control

**Choose isolated for MVP, migrate to shared for scale.**

---

### Advanced Topics

**Q: How would you add real-time collaborative editing of the graph?**

**A:** WebSocket-based architecture:

**1. WebSocket connection:**
```python
from fastapi import WebSocket

@app.websocket("/ws/{graph_id}")
async def websocket_endpoint(websocket: WebSocket, graph_id: str):
    await websocket.accept()
    
    # Subscribe to graph updates
    await subscribe_to_graph_changes(graph_id)
    
    while True:
        # Listen for changes
        change = await receive_graph_change()
        
        # Broadcast to all connected clients
        await websocket.send_json({
            "type": "node_updated",
            "node_id": change.node_id,
            "data": change.data
        })
```

**2. Operational Transformation (OT) or CRDT:**
- Handle concurrent edits (user A edits node 1, user B edits node 1)
- Use CRDT (Conflict-free Replicated Data Types) for automatic conflict resolution

**3. Change log:**
```sql
CREATE TABLE graph_changes (
    change_id UUID PRIMARY KEY,
    graph_id UUID,
    user_id UUID,
    change_type VARCHAR,  -- node_added, node_updated, edge_added, etc.
    change_data JSONB,
    timestamp TIMESTAMP,
    version INT
);
```

**4. Frontend sync:**
```javascript
const ws = new WebSocket('ws://localhost:8000/ws/graph123');

ws.onmessage = (event) => {
    const change = JSON.parse(event.data);
    if (change.type === 'node_updated') {
        updateNodeInGraph(change.node_id, change.data);
    }
};

// On local edit
function onNodeEdit(nodeId, newData) {
    ws.send(JSON.stringify({
        type: 'edit_node',
        node_id: nodeId,
        data: newData
    }));
}
```

---

**Q: How would you implement graph versioning (track changes over time)?**

**A:** Git-like version control for graphs:

**1. Graph snapshot storage:**
```python
class GraphVersion:
    version_id: str  # v1, v2, v3, ...
    parent_version: Optional[str]  # v1 → v2 (chain)
    graph_snapshot: NetworkXGraph
    changes: List[GraphChange]  # What changed from parent
    created_at: datetime
    created_by: str  # user_id
    commit_message: str

# Example changes
class GraphChange:
    change_type: str  # "node_added", "edge_removed", etc.
    node_id: Optional[str]
    edge_id: Optional[str]
    old_value: Any
    new_value: Any
```

**2. Version tree:**
```
v1 (initial PDF upload)
 ↓
v2 (user edits node A)
 ↓
v3 (user adds edge B→C)
 ├─ v4 (branch: experiment with different relations)
 └─ v5 (main: finalized version)
```

**3. Diff & merge:**
```python
def diff_graphs(v1: GraphVersion, v2: GraphVersion) -> List[GraphChange]:
    # Compare two graph versions
    changes = []
    
    # Nodes added
    new_nodes = set(v2.nodes) - set(v1.nodes)
    for node in new_nodes:
        changes.append(GraphChange("node_added", node_id=node))
    
    # Nodes removed
    removed_nodes = set(v1.nodes) - set(v2.nodes)
    # ... similar for edges, node properties, etc.
    
    return changes

def merge_graphs(base, v1, v2) -> GraphVersion:
    # Three-way merge (like git)
    # Resolve conflicts: keep both, prefer v1, prefer v2, manual
    ...
```

**4. UI for version control:**
- Timeline view of versions
- Diff visualization (red = removed, green = added)
- Revert to previous version
- Branch/merge UI

---

**Q: What metrics would you add for monitoring in production?**

**A:** Comprehensive observability:

**1. Application metrics (Prometheus):**
```python
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
request_count = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
request_duration = Histogram('http_request_duration_seconds', 'HTTP request duration')

# PDF processing
pdf_processing_time = Histogram('pdf_processing_seconds', 'Time to process PDF')
pdf_pages_processed = Counter('pdf_pages_total', 'Total PDF pages processed')

# Graph metrics
graph_nodes = Gauge('graph_nodes_total', 'Total nodes in graph')
graph_edges = Gauge('graph_edges_total', 'Total edges in graph')

# LLM metrics
llm_calls = Counter('llm_api_calls_total', 'LLM API calls', ['model', 'operation'])
llm_latency = Histogram('llm_latency_seconds', 'LLM API latency')
llm_tokens = Counter('llm_tokens_used', 'LLM tokens consumed')
llm_failures = Counter('llm_failures_total', 'LLM API failures')

# Agent metrics
agent_tool_usage = Counter('agent_tool_calls', 'Agent tool calls', ['tool_name'])
agent_query_time = Histogram('agent_query_seconds', 'Agent query time')
```

**2. Logging (structured logs with correlation IDs):**
```python
logger.info("PDF processing started", extra={
    "pdf_id": pdf_id,
    "user_id": user_id,
    "filename": filename,
    "correlation_id": request_id
})
```

**3. Tracing (OpenTelemetry):**
- Distributed tracing across microservices
- Trace request: PDF upload → extraction → embedding → graph building
- Identify slow steps in pipeline

**4. Alerts:**
```yaml
alerts:
  - name: HighLLMFailureRate
    condition: rate(llm_failures_total[5m]) > 0.05
    action: PagerDuty notification
  
  - name: SlowPDFProcessing
    condition: pdf_processing_seconds > 60
    action: Slack notification
  
  - name: HighMemoryUsage
    condition: memory_usage > 90%
    action: Auto-scale + alert
```

**5. Business metrics:**
- Total PDFs processed
- Active users (DAU, MAU)
- Average graphs per user
- Query success rate
- User satisfaction (explicit feedback)

---

## Key Takeaways for Interview

**Your unique selling points:**

1. **GraphRAG implementation** - Not just vector search, full knowledge graph
2. **Agent-based architecture** - LangGraph for intelligent tool orchestration
3. **Performance optimization** - Parallel processing, caching, deterministic
4. **Production-ready** - Error handling, retries, graceful degradation
5. **Scalable design** - Modular components, can swap FAISS→Pinecone, NetworkX→Neo4j

**Be ready to discuss:**
- Architecture decisions (why X over Y)
- Trade-offs (speed vs accuracy, simplicity vs features)
- Scalability (how to handle 10x, 100x, 1000x scale)
- Production concerns (monitoring, error handling, multi-tenancy)

**Demo talking points:**
1. Upload PDF → show parallel processing logs
2. Show graph visualization → explain undirected, 25 relation types
3. Click node → explain caching (2s first time, instant after)
4. Ask complex question → show agent using multiple tools
5. Ask follow-up → demonstrate conversation memory

**Strong answers:**
- "I chose X because [performance reason] and [scalability reason]"
- "The trade-off was [downside], but it's acceptable because [context]"
- "To scale to 10K PDFs, I would [specific architectural changes]"
- "I optimized [X] by [technique], achieving [quantitative improvement]"

Good luck! 🚀
