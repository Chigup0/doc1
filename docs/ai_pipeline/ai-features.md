### docs/usage/ai-features.md

# AI Features Guide

Zenith Drive incorporates advanced AI capabilities to enhance your file management experience.

## Contextual Chat

### Drive-Wide Chat

Ask questions about all files in your drive:

- "What's the total size of all my PDF documents?"
- "Find all files related to project Zenith"
- "Show me a summary of my financial documents"

### Folder-Level Chat

Restrict queries to specific folders:

- "What images are in the Vacation folder?"
- "Summarize the documents in the Work/Reports folder"

### Document-Specific Chat

Ask questions about individual files:

- "What are the key points in this PDF?"
- "Extract the main conclusions from this report"

## Highlight-Based Features

### Text Selection

Highlight any text in documents to:

- Get definitions of unfamiliar terms
- Receive explanations of complex concepts
- Request summaries of selected content

### Image Analysis

Select portions of images to:

- Identify objects and people
- Extract text via OCR
- Analyze charts and diagrams

**Technical Implementation - Multi-Modal Image Processing:**

```python
def process_uploaded_image(image_data: bytes, llm=None) -> Document:
    """Process uploaded image and return a Document with comprehensive analysis."""
    try:
        # Open image
        image = Image.open(io.BytesIO(image_data))

        # Perform comprehensive analysis
        analysis_results = image_processor.comprehensive_image_analysis(image, llm)

        # Create content for the document
        content_parts = []

        # Add OCR text if available
        if analysis_results['ocr'].get('full_text'):
            content_parts.append(f"OCR Text: {analysis_results['ocr']['full_text']}")

        # Add structure information
        if analysis_results['structure'].get('shapes'):
            shapes_info = f"Detected {analysis_results['structure']['total_shapes']} shapes and {analysis_results['structure']['total_lines']} lines"
            content_parts.append(shapes_info)

        # Add entity information if available
        if analysis_results.get('entities', {}).get('entities_detected'):
            content_parts.append(f"Entities: {analysis_results['entities']['entities_detected']}")

        # Add chart analysis if available
        if analysis_results.get('chart', {}).get('chart_analysis'):
            content_parts.append(f"Chart Analysis: {analysis_results['chart']['chart_analysis']}")

        content = '\n\n'.join(content_parts) if content_parts else analysis_results['summary']

        # Create document
        document = Document(
            page_content=content,
            metadata={
                'type': 'image_analysis',
                'analysis_results': analysis_results,
                'has_text': bool(analysis_results['ocr'].get('full_text')),
                'has_shapes': bool(analysis_results['structure'].get('shapes')),
                'has_entities': bool(analysis_results.get('entities', {}).get('entities_detected')),
                'image_size': analysis_results['image_info']['size'],
                'timestamp': analysis_results['timestamp']
            }
        )

        return document

    except Exception as e:
        logger.error(f"Image processing failed: {e}")
        return Document(
            page_content=f"Error processing image: {str(e)}",
            metadata={'type': 'image_error', 'error': str(e)}
        )
```

### Table Operations

Select table data to:

- Calculate aggregates (sum, average, etc.)
- Filter and sort data
- Create visualizations

## Semantic Search

Search based on meaning rather than just keywords:

- "Find documents about machine learning security"
- "Locate images from beach vacations"
- "Find spreadsheets with sales data"

**Technical Implementation - Hybrid RAG + GraphRAG Search:**

```python
def rag_answer(question: str, conversation_id: str = None, confidence_threshold: float = None) -> Dict[str, Any]:
    """
    Advanced semantic search with conversation awareness and confidence gating
    """
    if confidence_threshold is None:
        confidence_threshold = CONFIDENCE_THRESHOLD

    try:
        # Check if this is a follow-up question
        is_followup = False
        if conversation_id:
            is_followup = detect_followup_question(question, conversation_id)

        # Vector similarity search
        vector_docs = vector_db.similarity_search_with_score(question, k=10)
        confident_docs = [(doc, score) for doc, score in vector_docs if score >= confidence_threshold]

        if not confident_docs:
            return {
                "answer": "I couldn't find relevant information with sufficient confidence. Try being more specific.",
                "confidence": 0.0,
                "sources": [],
                "method": "insufficient_confidence"
            }

        # Graph-based contextual enhancement
        graph_context = ""
        if get_graph_db().is_connected():
            entities = extract_entities_simple(question)
            if entities:
                graph_facts = get_subgraph_facts(entities, max_facts=5)
                if graph_facts:
                    graph_context = f"\n\nKnowledge Graph Context:\n{graph_facts}"

        # Prepare context for LLM
        context_parts = []
        source_info = []

        for doc, confidence in confident_docs[:5]:  # Top 5 most confident results
            context_parts.append(f"Document: {doc.metadata.get('source', 'unknown')}\n{doc.page_content}")
            source_info.append({
                "source": doc.metadata.get('source', 'unknown'),
                "confidence": float(confidence),
                "chunk_id": doc.metadata.get('chunk_id', 'unknown')
            })

        full_context = "\n\n---\n\n".join(context_parts) + graph_context

        # Generate answer with conversation awareness
        conversation_prompt = ""
        if is_followup and conversation_id:
            conversation_prompt = f"\nNote: This appears to be a follow-up question in conversation {conversation_id}. Consider previous context appropriately."

        prompt = f"""Based on the following context, answer the question accurately and concisely.

Context:
{full_context}

Question: {question}{conversation_prompt}

Provide a helpful answer based on the context. If the context doesn't contain enough information, say so clearly."""

        # Generate response
        model = genai.GenerativeModel('gemini-1.5-flash')
        response = model.generate_content(prompt)

        # Calculate overall confidence
        avg_confidence = sum(conf for _, conf in confident_docs[:5]) / len(confident_docs[:5])

        return {
            "answer": response.text,
            "confidence": round(avg_confidence, 3),
            "sources": source_info,
            "method": "hybrid_rag_graphrag",
            "is_followup": is_followup,
            "graph_enhanced": bool(graph_context.strip())
        }

    except Exception as e:
        return {
            "answer": f"Error processing question: {e}",
            "confidence": 0.0,
            "sources": [],
            "method": "error"
        }

def detect_followup_question(question: str, conversation_id: str) -> bool:
    """
    Detect if a question is a follow-up in a conversation using AI
    """
    followup_indicators = [
        "what about", "and", "also", "furthermore", "additionally",
        "what", "how", "why", "when", "where", "which", "that",
        "it", "this", "these", "those", "they", "them"
    ]

    question_lower = question.lower()

    # Check for pronouns and follow-up indicators
    has_followup_words = any(indicator in question_lower for indicator in followup_indicators)

    # Check for short questions (likely follow-ups)
    is_short = len(question.split()) < 8

    # Check for missing context (pronouns without clear antecedents)
    pronouns = ["it", "this", "that", "they", "them", "these", "those"]
    has_pronouns = any(pronoun in question_lower.split() for pronoun in pronouns)

    return (has_followup_words and is_short) or has_pronouns
```

## Cross-Reference Features

When the AI references information from files:

1. Source citations are provided with links
2. Clicking a citation opens the source file
3. The relevant section is highlighted for context

## Knowledge Graph Visualization

View connections between your files:

- Visual representation of related documents
- Topic-based clustering of content
- Timeline views of document creation and modification

## API Integration

Developers can access AI features through our API:

```javascript
// Example: Querying drive context
const response = await fetch("/api/ai/query", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${token}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    query: "Summarize my project documents",
    context: "drive", // or "folder/:id", "file/:id"
    options: {
      include_citations: true,
      generate_visualization: false,
    },
  }),
});
```
