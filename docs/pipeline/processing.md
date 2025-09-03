# Constructing Knowledge Graphs: Major Functions

## Entity Extraction with LLM Prompts

**code snippet: Entity Extraction Prompt**

```python
ENTITY_EXTRACTION_PROMPT = """
You are a highly-specialized entity extraction model. Your task is to extract all salient entities from the provided text with confidence scores.

**Instructions:**
1. **Strict Adherence:** Analyze the entire text and extract every important entity.
2. **Canonical Form:** Present the entity name in its most standard form (e.g., "AI" → "Artificial Intelligence").
3. **Type Specificity:** Assign entity types: PERSON, ORGANIZATION, LOCATION, PRODUCT, TECHNOLOGY, EVENT, CONCEPT, TIME, NUMBER, MISCELLANEOUS.
4. **Attribute Richness:** Identify all relevant attributes and properties as a structured JSON object.
5. **Confidence Score:** Provide confidence (0.0-1.0) based on:
   - How clearly mentioned (0.9-1.0 for explicit mentions)
   - Relevance to main topic (0.7-0.9 for highly relevant)
   - Supporting context (0.5-0.7 for implied/context-dependent)

Return ONLY a JSON array with 'name', 'type', 'attributes', and 'confidence' fields.
Text: {text}
"""

def extract_entities(text: str) -> List[Dict[str, str]]:
    """Extract entities with types, attributes, and confidence scores using LLM."""
    try:
        llm_instance = get_llm()
        prompt = ENTITY_EXTRACTION_PROMPT.format(text=text)
        response = llm_instance.invoke(prompt).content.strip()

        entities = _safe_extract_json(response, list)
        if not entities:
            return []

        # Validate entities against text to reduce hallucinations
        validated_entities = []
        for entity in entities:
            if _validate_entity_in_text(entity.get('name', ''), text):
                validated_entities.append(entity)

        # Deduplicate similar entities
        return _deduplicate_entities(validated_entities)

    except Exception as e:
        print(f"[GraphRAG] Entity extraction failed: {e}")
        return []
```

## Confidence-Based Graph Operations

1. **Neo4jGraphRAG Class**
   - Manages Neo4j connection, creates constraints and indexes, and provides graph operations.
   - Ensures unique entity names and efficient graph queries via indexes.

**code snippet: Graph Database Initialization**

```python
class Neo4jGraphRAG:
    def __init__(self):
        self.driver = None
        self._connect()

    def _connect(self):
        """Initialize Neo4j connection."""
        try:
            self.driver = GraphDatabase.driver(NEO4J_URL, auth=(NEO4J_USER, NEO4J_PASSWORD))
            # Test connection
            with self.driver.session() as session:
                session.run("RETURN 1")
            print(f"[GraphRAG] Connected to Neo4j at {NEO4J_URL}")
            self._create_constraints()
        except Exception as e:
            print(f"[GraphRAG] Failed to connect to Neo4j: {e}")
            self.driver = None

    def _create_constraints(self):
        """Create necessary constraints and indexes."""
        with self.driver.session() as session:
            try:
                session.run("CREATE CONSTRAINT entity_name IF NOT EXISTS FOR (e:Entity) REQUIRE e.name IS UNIQUE")
                session.run("CREATE INDEX entity_type_idx IF NOT EXISTS FOR (e:Entity) ON (e.type)")
                session.run("CREATE INDEX document_file_idx IF NOT EXISTS FOR (d:Document) ON (d.file_id)")
                session.run("CREATE INDEX chunk_file_idx IF NOT EXISTS FOR (c:Chunk) ON (c.file_id)")
                print("[GraphRAG] Neo4j constraints and indexes created")
            except Exception as e:
                print(f"[GraphRAG] Warning: Could not create constraints/indexes: {e}")

    def close(self):
        """Close Neo4j connection."""
        if self.driver:
            self.driver.close()

    def is_connected(self):
        """Check if Neo4j is connected."""
        return self.driver is not None
```

2. **Entity Extraction & Validation**
   - `extract_entities(text)`: Uses LLM to extract canonical entities, types, attributes, and confidence scores from text.
   - `_validate_entity_in_text(entity_name, text)`: Ensures entities are present in the text to reduce hallucinations.
   - `_deduplicate_entities(entities)`: Merges similar entities and attributes to reduce noise.
   - `_calculate_similarity(str1, str2)`: Measures similarity for deduplication.
3. **Relation Extraction**
   - `extract_relations(text, entities)`: Uses LLM to extract relationships (subject, predicate, object, context, confidence) between entities in text.
   - Validates that both subject and object exist in the extracted entities or text.
4. **Graph Upsert and Linking**
   - `upsert_entities_and_relations(file_id, page, chunk_no, source, text)`: Extracts entities/relations from a chunk and stores them in Neo4j, linking documents, chunks, entities, and relationships with confidence scores.
   - Links document and chunk nodes, and connects entities and relations to their source chunks.
   - Supports dynamic graph updates as new files are ingested and existing files are modified.
5. **Query Entity Extraction**
   - `extract_query_entities(query)`: Uses LLM to extract key entities from a user query for targeted graph lookup.
6. **Subgraph Fact Retrieval**
   - `get_subgraph_facts(entities, file_id, folder_id, ...)`: Retrieves high-confidence facts and relationships from the graph relevant to the query entities, supporting multi-hop and fuzzy matching.
   - Uses exact, type-based, and partial/fuzzy matching strategies.

**code snippet: Intelligent Graph Querying**

```python
def get_subgraph_facts(
    entities: List[str],
    file_id: Optional[str] = None,
    folder_id: Optional[str] = None,
    max_facts: int = MAX_FACTS,
    min_confidence: float = 0.6
) -> str:
    """
    Build a textual block of 'facts' from the Neo4j graph, filtered by file_id and confidence.
    Uses intelligent matching: exact names, then types, then partial matches.
    Prioritizes high-confidence entities and relations to reduce noise.
    """
    if not entities or not get_graph_db().is_connected():
        return ""

    try:
        with get_graph_db().driver.session() as session:
            facts = []
            processed_entities = set()

            for query_entity in entities:
                query_entity_lower = query_entity.lower()

                # Strategy 1: Exact name match (case-insensitive) with confidence filtering
                entity_result = session.run("""
                    MATCH (e:Entity)
                    WHERE toLower(e.name) = $entity_name
                    AND (e.confidence IS NULL OR e.confidence >= $min_confidence)
                    OPTIONAL MATCH (c:Chunk)-[:MENTIONS]->(e)
                    WHERE ($file_id IS NULL OR c.file_id = $file_id)
                    RETURN e.name as name, e.type as type, e.attributes as attributes,
                           e.confidence as confidence,
                           collect(DISTINCT c.file_id) as mentioned_in_files
                    ORDER BY e.confidence DESC
                    LIMIT 5
                """, entity_name=query_entity_lower, file_id=file_id, min_confidence=min_confidence)

                found_entities = list(entity_result)

                # Strategy 2: If no exact match, try type-based matching
                if not found_entities:
                    # Map common query terms to types
                    type_mapping = {
                        'sets': 'SET', 'set': 'SET',
                        'functions': 'FUNCTION', 'function': 'FUNCTION',
                        'concepts': 'CONCEPT', 'concept': 'CONCEPT',
                        'locations': 'LOCATION', 'location': 'LOCATION'
                    }

                    target_type = type_mapping.get(query_entity_lower)
                    if target_type:
                        entity_result = session.run("""
                            MATCH (e:Entity)
                            WHERE e.type = $entity_type
                            AND (e.confidence IS NULL OR e.confidence >= $min_confidence)
                            OPTIONAL MATCH (c:Chunk)-[:MENTIONS]->(e)
                            WHERE ($file_id IS NULL OR c.file_id = $file_id)
                            RETURN e.name as name, e.type as type, e.attributes as attributes,
                                   e.confidence as confidence,
                                   collect(DISTINCT c.file_id) as mentioned_in_files
                            ORDER BY e.confidence DESC
                            LIMIT 5
                        """, entity_type=target_type, file_id=file_id, min_confidence=min_confidence)

                        found_entities = list(entity_result)
```

7. **Graph Statistics and Analytics**
   - `get_graph_stats()`: Returns counts and confidence metrics for entities, relationships, documents, and chunks in the graph.
   - Tracks entity types, average confidence, and high-confidence counts.
   - Supports document and chunk statistics for analytics.
8. **Graph Cleanup and Maintenance**
   - `cleanup_low_confidence_entities(confidence_threshold)`: Removes entities and relations below a confidence threshold to keep the graph high-quality.
   - `update_confidence_thresholds(entity_threshold, relation_threshold)`: Updates global thresholds for entity/relation filtering.
   - `clear_graph()`: Deletes all nodes and relationships from the Neo4j graph.

**code snippet: Confidence-Based Cleanup**

```python
def cleanup_low_confidence_entities(confidence_threshold: float = None) -> Dict:
    """
    Remove entities and relations below the confidence threshold to clean up the graph.
    """
    if not get_graph_db().is_connected():
        return {"error": "GraphRAG not connected"}

    if confidence_threshold is None:
        confidence_threshold = ENTITY_CONFIDENCE_THRESHOLD

    try:
        with get_graph_db().driver.session() as session:
            # Delete low-confidence entities
            entity_result = session.run("""
                MATCH (e:Entity)
                WHERE e.confidence < $threshold
                DETACH DELETE e
                RETURN count(e) as deleted_entities
            """, threshold=confidence_threshold)

            # Delete low-confidence relations
            relation_result = session.run("""
                MATCH ()-[r:RELATED_TO]-()
                WHERE r.confidence < $threshold
                DELETE r
                RETURN count(r) as deleted_relations
            """, threshold=confidence_threshold)

            deleted_entities = entity_result.single()["deleted_entities"]
            deleted_relations = relation_result.single()["deleted_relations"]

            return {
                "success": True,
                "deleted_entities": deleted_entities,
                "deleted_relations": deleted_relations,
                "threshold_used": confidence_threshold
            }

    except Exception as e:
        return {"error": f"Cleanup failed: {e}"}

def get_graph_stats() -> Dict:
    """Get comprehensive statistics about the knowledge graph."""
    if not get_graph_db().is_connected():
        return {"error": "GraphRAG not connected"}

    try:
        with get_graph_db().driver.session() as session:
            # Entity statistics
            entity_stats = session.run("""
                MATCH (e:Entity)
                RETURN
                    count(e) as total_entities,
                    avg(e.confidence) as avg_entity_confidence,
                    count(CASE WHEN e.confidence >= $high_conf THEN 1 END) as high_conf_entities,
                    collect(DISTINCT e.type) as entity_types
            """, high_conf=0.8).single()

            # Relation statistics
            relation_stats = session.run("""
                MATCH ()-[r:RELATED_TO]-()
                RETURN
                    count(r) as total_relations,
                    avg(r.confidence) as avg_relation_confidence,
                    count(CASE WHEN r.confidence >= $high_conf THEN 1 END) as high_conf_relations
            """, high_conf=0.8).single()

            # Document statistics
            doc_stats = session.run("""
                MATCH (d:Document)
                OPTIONAL MATCH (d)-[:CONTAINS]->(c:Chunk)
                RETURN
                    count(d) as total_documents,
                    count(c) as total_chunks
            """).single()

            return {
                "entities": {
                    "total": entity_stats["total_entities"],
                    "average_confidence": round(entity_stats["avg_entity_confidence"] or 0, 3),
                    "high_confidence_count": entity_stats["high_conf_entities"],
                    "types": entity_stats["entity_types"] or []
                },
                "relations": {
                    "total": relation_stats["total_relations"],
                    "average_confidence": round(relation_stats["avg_relation_confidence"] or 0, 3),
                    "high_confidence_count": relation_stats["high_conf_relations"]
                },
                "documents": {
                    "total": doc_stats["total_documents"],
                    "chunks": doc_stats["total_chunks"]
                },
                "thresholds": {
                    "entity_confidence": ENTITY_CONFIDENCE_THRESHOLD,
                    "relation_confidence": RELATION_CONFIDENCE_THRESHOLD
                }
            }

    except Exception as e:
        return {"error": f"Stats retrieval failed: {e}"}
```

9. **Constraint and Index Management**
   - `_create_constraints()`: Ensures unique entity names and efficient graph queries via indexes.
10. **Lazy Initialization and Connection Checking**
    - `get_graph_db()`: Provides a singleton instance of the graph database connection.
    - `is_connected()`: Checks if the Neo4j connection is active.
11. **Confidence-Based Filtering**
    - All major extraction and retrieval functions use confidence scores to filter and rank entities and relations, improving answer quality and reducing noise.
12. **Logging and Error Handling**
    - All functions include logging and error handling to ensure robustness and traceability.
13. **Multi-Hop and Fuzzy Matching**
    - Subgraph retrieval supports multi-hop reasoning and fuzzy matching for advanced queries.
14. **Dynamic Graph Updates**
    - Graph structure is updated as files are added, modified, or deleted, keeping knowledge up-to-date.
