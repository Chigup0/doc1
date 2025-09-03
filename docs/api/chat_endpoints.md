# RAG + Knowledge Graph (KAG) Architecture Documentation

## Overview

TheDrive implements a sophisticated hybrid retrieval system that combines **Retrieval-Augmented Generation (RAG)** with **Knowledge Graphs** to provide intelligent document search and conversational AI capabilities. This system processes user documents through multiple layers of analysis to create both vector embeddings and structured knowledge representations.

## System Architecture

### High-Level Components

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[React/Next.js UI]
        Chat[Chat Interface]
        Files[File Management]
    end
    
    subgraph "API Layer"
        Backend[FastAPI Backend]
        RagAPI[GraphRAG API]
    end
    
    subgraph "Processing Layer"
        Ingestion[Document Ingestion]
        Chunking[Text Chunking]
        Embedding[Vector Embedding]
        GraphExtraction[Entity/Relation Extraction]
    end
    
    subgraph "Storage Layer"
        S3[AWS S3<br/>File Storage]
        Postgres[PostgreSQL<br/>User/File Metadata]
        Chroma[ChromaDB<br/>Vector Store]
        Neo4j[Neo4j<br/>Knowledge Graph]
    end
    
    UI --> Backend
    Chat --> RagAPI
    Files --> Backend
    Backend --> S3
    Backend --> Postgres
    RagAPI --> Ingestion
    Ingestion --> Chunking
    Chunking --> Embedding
    Chunking --> GraphExtraction
    Embedding --> Chroma
    GraphExtraction --> Neo4j
```

## Document Ingestion Pipeline

### 1. File Upload & Storage Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend
    participant S3
    participant Database
    participant RAG_API
    
    User->>Frontend: Upload Document
    Frontend->>Backend: POST /upload
    Backend->>S3: Store File
    S3-->>Backend: File URL
    Backend->>Database: Save File Metadata
    Database-->>Backend: File ID
    Backend->>RAG_API: Trigger Ingestion
    RAG_API-->>Backend: Ingestion Started
    Backend-->>Frontend: Upload Success
    Frontend-->>User: File Uploaded
```

### 2. Document Processing Pipeline

```mermaid
graph LR
    subgraph "Document Ingestion"
        A[Raw Document] --> B[File Type Detection]
        B --> C[Content Extraction]
        C --> D[Text Preprocessing]
    end
    
    subgraph "Parallel Processing"
        D --> E[Text Chunking]
        D --> F[Entity Extraction]
        
        E --> G[Vector Embedding]
        F --> H[Relation Extraction]
        F --> I[Entity Linking]
        
        G --> J[Store in ChromaDB]
        H --> K[Store in Neo4j]
        I --> K
    end
    
    subgraph "Confidence Validation"
        K --> L[Confidence Scoring]
        L --> M[Quality Filtering]
        M --> N[Graph Cleanup]
    end
```

## RAG Implementation

### Traditional RAG Process

```mermaid
graph TD
    A[User Query] --> B[Query Preprocessing]
    B --> C[Vector Embedding]
    C --> D[Similarity Search in ChromaDB]
    D --> E[Retrieve Top-K Chunks]
    E --> F[Context Assembly]
    F --> G[LLM Prompt Construction]
    G --> H[Gemini API Call]
    H --> I[Generated Response]
    I --> J[Response Post-processing]
    J --> K[Return to User]
```

### Enhanced KAG Process

```mermaid
graph TD
    A[User Query] --> B[Query Analysis]
    B --> C[Entity Extraction from Query]
    C --> D[Parallel Retrieval]
    
    subgraph "Dual Retrieval System"
        D --> E[Vector Search in ChromaDB]
        D --> F[Graph Traversal in Neo4j]
        
        E --> G[Text Chunks]
        F --> H[Graph Facts]
    end
    
    G --> I[Context Enrichment]
    H --> I
    I --> J[Structured Context Assembly]
    J --> K[Enhanced LLM Prompt]
    K --> L[Gemini API with Rich Context]
    L --> M[Comprehensive Response]
```

## Knowledge Graph Construction

### Entity and Relation Extraction

```mermaid
graph LR
    subgraph "Text Processing"
        A[Document Chunk] --> B[Named Entity Recognition]
        B --> C[Relation Extraction]
        C --> D[Entity Linking]
    end
    
    subgraph "Graph Construction"
        D --> E[Entity Nodes Creation]
        D --> F[Relationship Edges Creation]
        E --> G[Property Assignment]
        F --> H[Confidence Scoring]
    end
    
    subgraph "Quality Assurance"
        G --> I[Validation Rules]
        H --> I
        I --> J[Confidence Thresholds]
        J --> K[Graph Storage in Neo4j]
    end
```

### Graph Schema

```cypher
// Example Neo4j Schema
(:Document {id, name, type, owner_id})
(:Entity {name, type, confidence, embedding_vector})
(:Person {name, title, organization})
(:Organization {name, industry, location})
(:Concept {name, definition, category})

// Relationships
(:Entity)-[:MENTIONED_IN]->(:Document)
(:Person)-[:WORKS_FOR]->(:Organization)
(:Entity)-[:RELATED_TO {confidence, type}]->(:Entity)
(:Document)-[:CONTAINS]->(:Entity)
```

## Query Processing Flow

### Step-by-Step Query Resolution

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant RAG_API
    participant LLM
    participant ChromaDB
    participant Neo4j
    
    User->>Frontend: Ask Question
    Frontend->>RAG_API: Query Request
    RAG_API->>RAG_API: Query Analysis
    RAG_API->>LLM: Extract Query Entities
    LLM-->>RAG_API: Entity List
    
    par Vector Search
        RAG_API->>ChromaDB: Similarity Search
        ChromaDB-->>RAG_API: Relevant Chunks
    and Graph Search
        RAG_API->>Neo4j: Graph Traversal
        Neo4j-->>RAG_API: Connected Facts
    end
    
    RAG_API->>RAG_API: Context Assembly
    RAG_API->>LLM: Enhanced Prompt
    LLM-->>RAG_API: Generated Answer
    RAG_API-->>Frontend: Response + Citations
    Frontend-->>User: Final Answer
```

## Context Enrichment Strategy

### Information Layering

```mermaid
graph TB
    subgraph "Context Assembly"
        A[Raw Query] --> B[Entity Extraction]
        B --> C[Graph Fact Retrieval]
        B --> D[Vector Similarity Search]
        
        C --> E[Structured Knowledge]
        D --> F[Unstructured Text Chunks]
        
        E --> G[Context Layering]
        F --> G
        
        G --> H[Final Context]
    end
    
    subgraph "Context Structure"
        H --> I[Graph Facts Section]
        H --> J[Raw Text Section]
        H --> K[Conversation Context]
        
        I --> L[Structured Knowledge Facts]
        J --> M[Source Document Chunks]
        K --> N[Previous Q&A History]
    end
```

## File-Vector DB Mapping System

### Re-ingestion Feature Flow

```mermaid
graph TD
    A[User Login] --> B[Check Ingestion Status API]
    B --> C[Query Database for File Status]
    C --> D{Files Need Processing?}
    
    D -->|Yes| E[Show Re-ingestion Button]
    D -->|No| F[Hide Button]
    
    E --> G[User Clicks Button]
    G --> H[Confirm Re-ingestion]
    H --> I[Start Background Processing]
    I --> J[Disable Chat Interface]
    J --> K[Process Pending Files]
    K --> L[Update File Status]
    L --> M[Poll Status Updates]
    M --> N{All Files Processed?}
    
    N -->|No| M
    N -->|Yes| O[Re-enable Chat]
    O --> P[Hide Button]
```

### Database Schema for File Tracking

```sql
-- FileSystemItem table tracks ingestion status
CREATE TABLE filesystem_items (
    id VARCHAR PRIMARY KEY,
    name VARCHAR NOT NULL,
    type VARCHAR NOT NULL, -- 'file' or 'folder'
    owner_id INTEGER REFERENCES users(id),
    parent_id VARCHAR REFERENCES filesystem_items(id),
    s3_key VARCHAR, -- For files only
    mime_type VARCHAR,
    size_bytes BIGINT,
    ingestion_status VARCHAR, -- 'pending', 'processing', 'completed', 'failed'
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

## Confidence-Based Quality Control

### Entity Confidence Scoring

```mermaid
graph LR
    subgraph "Confidence Metrics"
        A[Entity Extraction] --> B[LLM Confidence Score]
        A --> C[Named Entity Recognition Score]
        A --> D[Context Relevance Score]
        A --> E[Cross-Reference Validation]
    end
    
    subgraph "Scoring Algorithm"
        B --> F[Weighted Average]
        C --> F
        D --> F
        E --> F
        F --> G[Final Confidence Score]
    end
    
    subgraph "Quality Filtering"
        G --> H{Score >= Threshold?}
        H -->|Yes| I[Include in Graph]
        H -->|No| J[Exclude from Graph]
    end
```

### Graph Cleanup Process

```mermaid
graph TD
    A[Periodic Cleanup] --> B[Identify Low-Confidence Entities]
    B --> C[Check Entity Usage]
    C --> D{Used in Queries?}
    
    D -->|Yes| E[Increase Confidence]
    D -->|No| F{Below Minimum Threshold?}
    
    F -->|Yes| G[Remove Entity]
    F -->|No| H[Keep Entity]
    
    G --> I[Update Connected Relations]
    E --> J[Maintain Entity]
    H --> J
    I --> J
```

## API Endpoints

### Key RAG API Endpoints

```yaml
# Document Ingestion
POST /ingest/file
  - Upload and process document
  - Returns: ingestion_id, status

# Query Processing
POST /query/stream
  - Streaming RAG query with SSE
  - Returns: Server-Sent Events stream

GET /query
  - Non-streaming RAG query
  - Returns: answer, citations, confidence

# Graph Management
GET /graph/stats
  - Graph statistics and health
  - Returns: node_count, relationship_count, confidence_distribution

POST /graph/cleanup
  - Clean low-confidence entities
  - Returns: removed_count, updated_count

# Status Checking
GET /drive/check-ingestion-status
  - Check user files needing ingestion
  - Returns: needs_ingestion, files_count, is_active

POST /drive/reingest-files
  - Trigger re-ingestion of pending files
  - Returns: processed_count, failed_files
```

## Performance Optimizations

### Caching Strategy

```mermaid
graph LR
    subgraph "Multi-Level Caching"
        A[User Query] --> B[Query Cache Check]
        B --> C{Cache Hit?}
        
        C -->|Yes| D[Return Cached Result]
        C -->|No| E[Process Query]
        
        E --> F[Vector Cache Check]
        F --> G[Graph Cache Check]
        G --> H[LLM Processing]
        H --> I[Cache Result]
        I --> J[Return Response]
    end
```

### Parallel Processing

```mermaid
graph TB
    subgraph "Concurrent Operations"
        A[Query Received] --> B[Split Processing]
        
        B --> C[Vector Search Thread]
        B --> D[Graph Search Thread]
        B --> E[Entity Extraction Thread]
        
        C --> F[ChromaDB Query]
        D --> G[Neo4j Query]
        E --> H[LLM Entity Analysis]
        
        F --> I[Result Aggregation]
        G --> I
        H --> I
        
        I --> J[Context Assembly]
        J --> K[Final Response]
    end
```

## Configuration Settings

### Environment Variables

```env
# GraphRAG Configuration
ENABLE_GRAPHRAG=1
ENTITY_CONFIDENCE_THRESHOLD=0.6
RELATION_CONFIDENCE_THRESHOLD=0.7
ENABLE_EMBEDDING_VALIDATION=1

# Vector Database
CHROMA_URL=http://chroma:8000
COLLECTION=thedrive

# Graph Database
NEO4J_URL=bolt://neo4j:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password123

# LLM Configuration
GEMINI_API_KEY=your_api_key
MODEL_NAME=gemini-1.5-pro
RESPONSE_MODEL=gemini-2.5-pro

# Context Limits
MAX_CONTEXT_CHARS=12000
```

## Monitoring and Analytics

### System Health Metrics

```mermaid
graph LR
    subgraph "Performance Metrics"
        A[Query Response Time]
        B[Vector Search Latency]
        C[Graph Query Performance]
        D[LLM API Response Time]
    end
    
    subgraph "Quality Metrics"
        E[Answer Relevance Score]
        F[Citation Accuracy]
        G[Graph Fact Precision]
        H[User Satisfaction Rating]
    end
    
    subgraph "System Metrics"
        I[Database Connection Pool]
        J[Memory Usage]
        K[Storage Utilization]
        L[API Error Rates]
    end
```

## Benefits of the Hybrid Approach

### RAG + KAG Advantages

1. **Enhanced Context Understanding**
   - Vector search provides semantic similarity
   - Knowledge graph provides structured relationships
   - Combined approach offers comprehensive context

2. **Improved Answer Quality**
   - Factual accuracy through structured knowledge
   - Contextual relevance through vector similarity
   - Reduced hallucination through grounded facts


