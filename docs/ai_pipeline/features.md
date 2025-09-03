# Key Features

## Multi-File Format Support

- **Documents:** Supports PDF, DOCX, TXT, PPTX. Each format is parsed using specialized loaders, with metadata extraction and layout preservation.
- **Spreadsheets:** Handles CSV and XLSX, with intelligent chunking and column/row summarization for semantic search.
- **Images:** JPG, PNG, and other formats are processed for text (OCR), diagrams, charts, and entity detection. Visual content is indexed and made searchable.
- **Hybrid Ingestion:** Automatically detects file type and applies the optimal processing pipeline, including multimodal analysis for files containing both text and images.

## RAG (Retrieval-Augmented Generation) Functionality

- **Universal Ingestion:** Files are detected and loaded using format-specific strategies, supporting batch and incremental uploads.
- **Chunking:** Documents are split into context-aware chunks, with parameters tuned for each file type to maximize retrieval quality.
- **Semantic Embeddings:** Chunks are embedded using Google Gemini models, enabling meaning-based search and retrieval.
- **Graph-Based Knowledge:** Entities and relationships are extracted from text and images using LLMs, then stored in a Neo4j graph database for advanced querying and analytics.
- **Multimodal Support:** Images are analyzed for printed and handwritten text (Tesseract, EasyOCR), diagrams, charts, and entities using vision models. Results are combined and deduplicated for comprehensive coverage.

## Intelligent Chat & Search

- **Drive-Level Chat:** Users can ask questions spanning all files; answers are grounded in indexed content and cite sources, with context-aware retrieval.
- **Folder/Document-Level Chat:** Restricts context to the selected folder or document, ensuring privacy and relevance for targeted queries.
- **Highlight-Based AI Popups:** Users can highlight text, table regions, or image areas for instant explanations, definitions, or metrics, powered by LLMs and vision models.
- **Semantic Search:** Supports meaning-based search across all content, including cross-file and cross-modal queries.
- **Cross-Referencing:** Answers include citations and direct links to file locations; opening a file highlights the relevant chunk for user convenience.
- **Knowledge Visualization:** Relationships and connections are visualized as interactive knowledge graphs, supporting explainability and analytics.

**code snippet: Conversation-Aware RAG**

```python
def detect_followup_question(current_query: str, conversation_history: List[Dict]) -> Dict:
    """
    Detect if the current query is a follow-up to previous conversation.
    Returns analysis with follow-up detection and context preservation recommendation.
    """
    if not conversation_history:
        return {
            "is_followup": False,
            "confidence": 0.0,
            "reasoning": "No previous conversation",
            "should_preserve_context": False
        }

    # Get the most recent conversation turn
    last_turn = conversation_history[-1]
    prev_query = last_turn.get("query", "")
    prev_response = last_turn.get("response", "")[:500]  # Limit response length

    # Simple heuristic checks first
    current_lower = current_query.lower()

    # Check for obvious follow-up indicators
    followup_indicators = [
        "what about", "how about", "tell me more", "can you explain", "expand on",
        "what does that mean", "what is that", "what are they", "what is it",
        "more details", "elaborate", "clarify", "continue", "also",
        "in addition", "furthermore", "what else", "anything else"
    ]

    pronoun_indicators = [
        " it ", " this ", " that ", " they ", " them ", " these ", " those ",
        "the above", "mentioned", "previous", "earlier"
    ]

    # Quick heuristic scoring
    heuristic_score = 0.0
    reasons = []

    for indicator in followup_indicators:
        if indicator in current_lower:
            heuristic_score += 0.3
            reasons.append(f"Contains follow-up phrase: '{indicator}'")

    for pronoun in pronoun_indicators:
        if pronoun in current_lower:
            heuristic_score += 0.4
            reasons.append(f"Contains reference pronoun: '{pronoun.strip()}'")

    # Check if query is very short (likely a follow-up)
    if len(current_query.split()) <= 5 and heuristic_score > 0:
        heuristic_score += 0.2
        reasons.append("Short query with follow-up indicators")

    # Check for question words in short queries
    question_words = ["what", "how", "why", "when", "where", "which", "who"]
    if any(current_lower.startswith(qw) for qw in question_words) and len(current_query.split()) <= 8:
        heuristic_score += 0.1
        reasons.append("Short question likely referencing previous context")

    # If heuristic score is high enough, consider it a follow-up
    if heuristic_score >= 0.4:
        return {
            "is_followup": True,
            "confidence": min(0.9, heuristic_score),
            "reasoning": "; ".join(reasons),
            "should_preserve_context": True
        }

    # For borderline cases, use LLM analysis
    if heuristic_score > 0.1 or len(current_query.split()) <= 6:
        try:
            prompt = FOLLOW_UP_DETECTION_PROMPT.format(
                prev_query=prev_query,
                prev_response=prev_response,
                current_query=current_query
            )

            llm_instance = llm()
            response = llm_instance.invoke(prompt).content.strip()

            # Parse LLM response
            llm_analysis = _safe_extract_json(response, dict)
            if llm_analysis:
                return llm_analysis
        except Exception as e:
            pass

    return {
        "is_followup": False,
        "confidence": 1.0 - heuristic_score,
        "reasoning": "No clear follow-up indicators found",
        "should_preserve_context": False
    }
```

## Data Management & Cleanup

- **Complete File Deletion:** Removes files from both vector database and knowledge graph, ensuring no orphaned data.
- **Bulk Operations:** Supports batch deletion of multiple files with detailed success/failure tracking.
- **Orphaned Data Detection:** Identifies and cleans up data that may have been orphaned due to external file deletions.
- **Preview Operations:** Allows users to preview what will be deleted before actual deletion.

**code snippet: Complete File Deletion**

```python
def delete_file_completely(file_id: str) -> Dict:
    """
    Delete a file completely from both vector database and knowledge graph.
    """
    result = {
        "file_id": file_id,
        "vector_db": {"success": False},
        "knowledge_graph": {"success": False},
        "overall_success": False
    }

    # Delete from vector database
    vector_result = delete_file_from_vector_db(file_id)
    result["vector_db"] = vector_result

    # Delete from knowledge graph
    graph_result = delete_file_from_knowledge_graph(file_id)
    result["knowledge_graph"] = graph_result

    # Overall success if at least one deletion succeeded
    result["overall_success"] = vector_result["success"] or graph_result["success"]

    return result

def cleanup_orphaned_entities() -> Dict:
    """
    Clean up entities that have no mention relationships from any chunks.
    This should be called after deleting files to clean up orphaned entities.
    """
    result = {
        "success": False,
        "deleted_entities": 0,
        "deleted_relations": 0,
        "errors": []
    }

    graphrag_db = _get_graphrag_db()
    if not graphrag_db or not graphrag_db.is_connected():
        result["errors"].append("GraphRAG not available or not connected")
        return result

    try:
        with graphrag_db.driver.session() as session:
            # First, delete all RELATES relationships connected to orphaned entities
            orphaned_relations_query = """
            MATCH (e:Entity)
            WHERE NOT EXISTS((:Chunk)-[:MENTIONS]->(e))
            MATCH (e)-[r:RELATES]-(other)
            DELETE r
            RETURN count(r) as deleted_relations
            """
            relations_result = session.run(orphaned_relations_query).single()
            deleted_relations = relations_result["deleted_relations"] if relations_result else 0

            # Then delete the orphaned entities themselves
            cleanup_query = """
            MATCH (e:Entity)
            WHERE NOT EXISTS((:Chunk)-[:MENTIONS]->(e))
            DELETE e
            RETURN count(e) as deleted_entities
            """
            cleanup_result = session.run(cleanup_query).single()
            result["deleted_entities"] = cleanup_result["deleted_entities"] if cleanup_result else 0
            result["deleted_relations"] = deleted_relations
            result["success"] = True

            print(f"[GraphRAG] Cleaned up {result['deleted_entities']} orphaned entities and {deleted_relations} orphaned relationships")

    except Exception as e:
        result["errors"].append(f"GraphRAG error: {str(e)}")
        print(f"[GraphRAG] Failed to cleanup orphaned entities: {e}")

    return result
```

## Performance Optimization

### Caching Strategy

- **Vector embeddings**: Cached to avoid recomputation
- **Graph queries**: Intelligent caching of subgraph results
- **LLM responses**: Context-aware response caching

### Batch Processing

- **Bulk ingestion**: Efficient processing of multiple files
- **Parallel extraction**: Concurrent entity/relation extraction
- **Optimized indexing**: Smart chunking and vector generation

**code snippet: Performance Monitoring & Health Checks**

```python
import time
from functools import wraps
from concurrent.futures import ThreadPoolExecutor, as_completed

def performance_monitor(func):
    """Decorator to monitor function performance"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        try:
            result = func(*args, **kwargs)
            execution_time = time.time() - start_time

            # Add performance data to result if it's a dict
            if isinstance(result, dict):
                result['_performance'] = {
                    'execution_time': round(execution_time, 3),
                    'function': func.__name__
                }

            return result

        except Exception as e:
            execution_time = time.time() - start_time
            print(f"Function {func.__name__} failed after {execution_time:.3f}s: {e}")
            raise

    return wrapper

def get_system_health() -> Dict[str, Any]:
    """Comprehensive system health check"""
    health_status = {
        "timestamp": time.time(),
        "overall_status": "healthy",
        "components": {}
    }

    # Check vector database
    try:
        test_query = vector_db.similarity_search("test", k=1)
        health_status["components"]["vector_db"] = {
            "status": "healthy",
            "response_time": "< 100ms"
        }
    except Exception as e:
        health_status["components"]["vector_db"] = {
            "status": "unhealthy",
            "error": str(e)
        }
        health_status["overall_status"] = "degraded"

    # Check graph database
    if get_graph_db().is_connected():
        try:
            with get_graph_db().driver.session() as session:
                result = session.run("RETURN 1 as test").single()
                health_status["components"]["graph_db"] = {
                    "status": "healthy",
                    "connection": "active"
                }
        except Exception as e:
            health_status["components"]["graph_db"] = {
                "status": "unhealthy",
                "error": str(e)
            }
            health_status["overall_status"] = "degraded"
    else:
        health_status["components"]["graph_db"] = {
            "status": "disconnected",
            "connection": "inactive"
        }
        health_status["overall_status"] = "degraded"

    return health_status
```

## System Reliability

### Error Handling

- **Graceful degradation**: System continues with reduced functionality
- **Automatic retries**: Smart retry logic for transient failures
- **Fallback mechanisms**: Alternative processing methods when primary fails

### Monitoring and Logging

- **Performance tracking**: Response times and resource usage
- **Error reporting**: Detailed error logs with context
- **Health checks**: Continuous monitoring of system components
