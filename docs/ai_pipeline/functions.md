# RAG Pipeline: Key Functions and Workflow

1. **File Type Detection**
   - `detect_file_type(file_path, content_type)`: Determines file type using extension and MIME type. Ensures robust handling of PDFs, DOCX, TXT, CSV, images, and more. This step enables the pipeline to select the correct loader and chunking strategy.

**code snippet:**

```python
def detect_file_type(file_path: str, content_type: Optional[str] = None) -> str:
    """Detect file type from extension and MIME type."""
    path = Path(file_path)
    extension = path.suffix.lower()

    # Primary detection by extension
    if extension == '.pdf':
        return 'pdf'
    elif extension in ['.txt', '.md', '.rst']:
        return 'text'
    elif extension in ['.docx', '.doc']:
        return 'docx'
    elif extension == '.csv':
        return 'csv'
    elif extension in ['.jpeg', '.jpg', '.png', '.gif', '.bmp', '.webp', '.tiff', '.tif']:
        return 'image'

    # Fallback to MIME type
    if content_type:
        if 'pdf' in content_type:
            return 'pdf'
        elif 'text' in content_type:
            return 'text'
        elif 'spreadsheet' in content_type or 'csv' in content_type:
            return 'csv'
        elif 'word' in content_type or 'document' in content_type:
            return 'docx'
        elif 'image' in content_type:
            if 'jpeg' in content_type or 'jpg' in content_type:
                return 'jpg'
            elif 'png' in content_type:
                return 'png'
            elif 'gif' in content_type:
                return 'gif'
```

2. **Specialized Loading**

   - `load_pdf(path)`, `load_text(path)`, `load_docx(path)`, `load_csv(path)`: Each loader parses its format, extracts metadata, and prepares content for chunking. For images, `process_uploaded_image(image_data, llm)` applies OCR, diagram, and entity analysis.

3. **Chunking**

   - `chunk_docs(docs, file_type, chunk_size, overlap)`: Splits loaded documents into context-rich chunks. Chunk size and overlap are tuned per file type (e.g., larger for CSVs, structured for DOCX/PDF) to maximize retrieval quality and context preservation.

4. **Embedding & Vector Store**

   - `embedder()`: Returns the embedding model (Google Gemini) for semantic search.
   - `get_vs()`, `get_cache_vs()`: Access the main and cache Chroma vector stores. Chunks are embedded and stored for fast semantic retrieval and caching.

5. **Graph Construction (Optional)**

   - `get_graphrag_functions()`: Loads GraphRAG functions for entity/relation extraction and graph updates. Entities and relationships are extracted from text and images, then stored in Neo4j for advanced cross-file reasoning.

6. **Image & Multimodal Analysis**
   - `process_uploaded_image(image_data, llm)`: Performs OCR (Tesseract, EasyOCR), diagram structure analysis, chart/graph analysis, and entity detection. Results are deduplicated and combined for comprehensive coverage.
   - `get_image_summary(image_bytes, llm)`: Uses a vision-capable LLM to generate descriptive summaries for images, including objects, text, and visual features.

**code snippet: Image Processing Pipeline**

```python
class ImageProcessor:
    """Advanced image processing for OCR and visual understanding."""

    def __init__(self):
        # Initialize EasyOCR reader for handwritten text
        self.ocr_reader = easyocr.Reader(['en'])

    def preprocess_image_for_ocr(self, image: Image.Image) -> Image.Image:
        """Preprocess image to improve OCR accuracy."""
        # Convert to grayscale
        if image.mode != 'L':
            image = image.convert('L')

        # Enhance contrast
        enhancer = ImageEnhance.Contrast(image)
        image = enhancer.enhance(2.0)

        # Apply sharpening filter
        image = image.filter(ImageFilter.SHARPEN)

        # Convert to OpenCV format for advanced processing
        opencv_image = cv2.cvtColor(np.array(image), cv2.COLOR_GRAY2BGR)

        # Apply morphological operations to clean up text
        kernel = np.ones((1,1), np.uint8)
        opencv_image = cv2.morphologyEx(opencv_image, cv2.MORPH_CLOSE, kernel)

        # Convert back to PIL
        processed_image = Image.fromarray(cv2.cvtColor(opencv_image, cv2.COLOR_BGR2GRAY))
        return processed_image

    def extract_text_comprehensive(self, image: Image.Image) -> Dict[str, Union[str, List[Dict]]]:
        """Extract text using both OCR methods and combine results."""
        tesseract_result = self.extract_text_tesseract(image)
        easyocr_result = self.extract_text_easyocr(image)

        # Combine results - prioritize EasyOCR for handwritten text detection
        combined_text_blocks = []
        combined_text = ""

        # Use EasyOCR results as primary
        if easyocr_result['text_blocks']:
            combined_text_blocks.extend(easyocr_result['text_blocks'])
            combined_text += easyocr_result['full_text'] + " "

        # Add Tesseract results if they provide additional information
        if tesseract_result['text_blocks']:
            # Simple deduplication - avoid adding very similar text
            for block in tesseract_result['text_blocks']:
                is_duplicate = any(
                    self._text_similarity(block['text'], existing['text']) > 0.8
                    for existing in combined_text_blocks
                )
                if not is_duplicate:
                    combined_text_blocks.append({**block, 'method': 'tesseract'})
                    combined_text += block['text'] + " "

        return {
            'full_text': combined_text.strip(),
            'text_blocks': combined_text_blocks,
            'method': 'comprehensive'
        }
```

7. **Universal Ingestion**

   - `ingest_file(path, file_id, folder_id, file_type)`: Orchestrates detection, loading, chunking, embedding, and storage. Handles multimodal files and updates the knowledge graph if enabled.

8. **Retrieval & Answer Generation**
   - `_rewrite_query(q)`: Refines user queries for better retrieval.
   - `_search_docs_with_scores(vs, query, where, k)`: Retrieves relevant chunks from the vector store, using semantic similarity and filtering by scope.
   - `rag_answer(query, scope, folder_id, file_id, k)`: Main function for answering queries—retrieves context, generates answers with LLM, and provides citations and confidence scores.

**code snippet: Query Processing with Confidence**

```python
def rag_answer(query: str, scope: str = "drive", folder_id: Optional[str] = None,
               file_id: Optional[str] = None, k: int = 10) -> Dict:
    """
    Non-streaming answer. (Your UI can still prefer SSE.)
    Includes namespaced cache and confidence gating.
    """
    vs = get_vs()
    cache_vs = get_cache_vs()

    # Auto-detect file scope based on query content
    scope, file_id = _auto_detect_file_scope(query, vs, scope, file_id)

    # scope filter
    where: Dict[str, str] = {}
    if scope == "folder" and folder_id:
        where["folder_id"] = folder_id
    if scope == "file" and file_id:
        where["file_id"] = file_id

    print(f"[RAG] rag_answer called with scope={scope}, file_id={file_id}, folder_id={folder_id}")
    print(f"[RAG] Where filter: {where}")

    # namespaced cache key/query
    cache_query = f"scope={scope}|folder={folder_id}|file={file_id}|q={query}"
    try:
        ch = cache_vs.similarity_search_with_score(cache_query, k=1)
        if ch:
            doc, dist = ch[0]
            if dist <= CACHE_DISTANCE_MAX:
                meta = doc.metadata or {}
                if "answer_json" in meta:
                    return json.loads(meta["answer_json"])
    except Exception:
        pass

    # retrieval
    q_lower = query.lower()
    if ("summarize" in q_lower or "summary" in q_lower) and (scope == "file" and file_id):
        rewritten_query = query
        pairs = _search_docs_with_scores(vs, "summary", where, k=max(k, 30))
    else:
        rewritten_query = _rewrite_query(query)
        kw = " ".join(_keywords(query))
        boosted = f"{rewritten_query} {kw}".strip()
        pairs = _search_docs_with_scores(vs, boosted, where, k=k)

    docs = [d for d, _ in pairs]
    distances = [dist for _, dist in pairs]
    avg_distance = (sum(distances)/len(distances)) if distances else 1.0
    confidence = round(_confidence_from_distances(distances), 3)

    if not docs or avg_distance > CONF_DISTANCE_MAX:
        return {
            "answer": "I don't know based on the indexed files.",
            "citations": [],
            "rewritten_query": rewritten_query,
            "confidence": confidence,
        }

    # build base context from vector hits
    ctx = _context_block(docs, settings.MAX_CONTEXT_CHARS)

    # prepend Graph Facts when enabled
    graphrag_funcs = get_graphrag_functions()
    if _GR_ENABLED and graphrag_funcs:
        try:
            ents = graphrag_funcs['extract_query_entities'](query)
            # Use confidence-based graph fact retrieval with minimum confidence filtering
            gctx = graphrag_funcs['get_subgraph_facts'](
                ents,
                file_id=file_id if scope == "file" else None,
                folder_id=folder_id if scope == "folder" else None,
                max_facts=settings.MAX_CONTEXT_CHARS // 100,  # Adjust based on context limit
                min_confidence=0.6  # Only include high-confidence facts
            )
            if gctx:
                ctx = f"# Graph Facts\n{gctx}\n---\n" + ctx
        except Exception:
            pass

    prompt = f"{SYSTEM_PROMPT}\n\n# Question\n{query}\n\n# Context\n{ctx}\n# Answer"
    answer_text = response_llm().invoke(prompt).content.strip()  # Use separate response model

    citations = []
    for d in docs:
        m = d.metadata or {}
        citations.append({
            "file_id": m.get("file_id"),
            "chunk_no": m.get("chunk_no"),
            "page": m.get("page"),
            "source": m.get("source"),
        })
```

9. **Conversation Awareness**
   - `detect_followup_question(current_query, conversation_history)`: Detects follow-up questions using heuristics and LLM analysis, preserving context for multi-turn conversations.
   - `conversation_aware_rag(query, conversation_history, scope, folder_id, file_id, k)`: Maintains context across turns, adjusts retrieval scope, and enhances query with conversation history for coherent answers.
