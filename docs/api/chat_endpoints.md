# Chat API for TheDrive

The **Chat API** in **TheDrive** enables users to interact with their files through intelligent queries, powered by **GraphRAG**. The API handles **chat-based queries**, performs **context validation**, and provides users with answers based on AI processing. It also checks the **status of AI processing** for each user.

---

## 1. Endpoints Overview

The following endpoints are available in the Chat API:

1. **POST /query**: Handles chat queries related to files and folders, using GraphRAG to provide intelligent answers.
2. **GET /status**: Provides the status of the user's chat readiness, including the number of files processed by AI.

---

## 2. Endpoint Details

### 2.1 POST /query

This endpoint handles user chat queries. The query is processed by the **GraphRAG** module, which generates a response based on the file context or AI processing.

#### Request Body:
A sample request body is like :
***json***
```
{
    "query": "What is the total revenue from last year's financial report?",
    "context": "file-12345"  // Optional: Can specify a file/folder context
}
```
query: The user's question or query.

context: The file or folder context in which the query is being asked (optional).

Response:

200 OK: Returns the answer to the query and the relevant sources.

500 Internal Server Error: If an error occurs during chat processing.

Example Response:
```
{
    "answer": "The total revenue from last year's financial report is $5 million.",
    "sources": [
        {
            "id": "community-1",
            "name": "Knowledge Cluster 1",
            "relevance": 0.95
        },
        {
            "id": "community-2",
            "name": "Knowledge Cluster 2",
            "relevance": 0.90
        }
    ]
}
```
**Code Example** :
```
@router.post("/query", response_model=schemas.ChatResponse)
async def handle_chat_query(
    query: schemas.ChatQuery,
    db: Session = Depends(database.get_db),
    current_user: models.User = Depends(get_current_user)
):
    """Handle chat queries with GraphRAG"""
    try:
        if not GRAPHRAG_AVAILABLE:
            # Fallback to mock response
            return {
                "answer": f"GraphRAG not available. Mock response for: '{query.query}'",
                "sources": []
            }
        
        # Validate context if specified
        if query.context and query.context != "drive":
            # Check if user owns the folder/file
            item = db.query(models.FileSystemItem).filter_by(
                id=query.context,
                owner_id=current_user.id
            ).first()
            if not item:
                raise HTTPException(status_code=404, detail="Context not found or access denied")
        
        # Get user's GraphRAG chatter
        chatter = await get_user_chatter(current_user.id)
        
        # Process the query
        result = await chatter.ask_question(query.query, "hybrid")
        
        if 'error' in result:
            raise HTTPException(status_code=500, detail=result['error'])
        
        # Format sources from GraphRAG result
        sources = []
        if result.get('graphrag_details'):
            # Extract source information from GraphRAG
            communities = result['graphrag_details'].get('selected_communities', [])
            for i, community in enumerate(communities[:3]):  # Top 3 sources
                sources.append({
                    "id": f"community-{i}",
                    "name": f"Knowledge Cluster {i+1}",
                    "relevance": result.get('confidence', 0.5)
                })
        
        return schemas.ChatResponse(
            answer=result['answer'],
            sources=sources
        )
        
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Chat processing error: {str(e)}")
```

2.2 GET /status

This endpoint retrieves the chat readiness status for a user, including the number of files that have been processed by AI. It helps users understand if their files are ready for interaction with GraphRAG.

Response:

- 200 OK: Returns the current status of chat readiness.

- 503 Service Unavailable: If GraphRAG service is not available.

Example Response:

```
{
    "ready": true,
    "message": "Chat ready with 5 processed files",
    "processed_files": 5,
    "total_files": 10,
    "processing_percentage": 50
}
```
Code Example:
```
@router.get("/status")
async def get_chat_status(
    db: Session = Depends(database.get_db),
    current_user: models.User = Depends(get_current_user)
):
    """Get chat readiness status for user"""
    if not GRAPHRAG_AVAILABLE:
        return {
            "ready": False,
            "message": "GraphRAG service not available",
            "processed_files": 0,
            "total_files": 0
        }
    
    # Count user's files and processing status
    files = db.query(models.FileSystemItem).filter_by(
        owner_id=current_user.id,
        type="file"
    ).all()
    
    total_files = len(files)
    processed_files = sum(1 for f in files if getattr(f, 'ai_processed', False))
    
    return {
        "ready": processed_files > 0,
        "message": f"Chat ready with {processed_files} processed files" if processed_files > 0 else "No files processed yet",
        "processed_files": processed_files,
        "total_files": total_files,
        "processing_percentage": (processed_files / total_files * 100) if total_files > 0 else 0
    }
```
3. GraphRAG Integration
3.1 What is GraphRAG?

GraphRAG is an AI-powered tool for processing and querying PDF documents and other types of data. It enables users to ask context-aware questions and receive answers by analyzing document contents, relationships, and AI-generated insights.

- PDFChatter: This is a module in GraphRAG that is used to process PDF documents and generate responses based on their content.

- Graph-Based Queries: The system uses graph-based algorithms to understand the relationships between entities within the documents and provide relevant answers.

3.2 Chat Handling

Each user has a GraphRAG chatter instance that interacts with documents and data specific to that user.

The PDFChatter processes files (if available) and returns relevant answers to the queries asked by users.

The system allows for context-specific queries, where users can ask questions related to a specific file or document.

3.3 Example Workflow

- User Query: A user asks, "What is the total revenue from last year's financial report?"

- Context Verification: If the query specifies a file context, the system ensures the user owns the file.

- GraphRAG Processing: The PDFChatter analyzes the document to extract relevant information.

- Response: The system returns the extracted answer and relevant sources from the document, if available.

4. Error Handling and Fallbacks

GraphRAG Not Available: If the GraphRAG service is not available, the system will provide a mock response indicating that GraphRAG is unavailable.

Context Not Found: If the specified context (e.g., a file ID) is invalid or the user does not own the file, an HTTP 404 error is returned.

Query Processing Error: Any unexpected error during query processing (e.g., failure in AI processing) will result in a 500 Internal Server Error with a detailed error message.

5. Conclusion

The Chat API in TheDrive enables users to interact with their files through contextual AI-powered queries, leveraging GraphRAG for document-based insights. By integrating GraphRAG, TheDrive provides a unique and intelligent way for users to retrieve information from their files. This system allows for both file-specific queries and semantic search, providing users with accurate, context-aware answers while ensuring smooth integration with AI processing and graph-based models.

The API is designed to handle errors gracefully and provide fallback responses when necessary, ensuring a seamless user experience even when the underlying AI service is unavailable.


---

### Key Sections:
1. **POST /query**: Handles AI-driven chat queries, processes context, and returns answers.
2. **GET /status**: Provides the status of the AI processing for the user’s files.
3. **GraphRAG Integration**: Explains how **GraphRAG** and **PDFChatter** are used to process user queries and provide insights.
4. **Error Handling**: Describes how the system manages errors, such as unavailable services or invalid contexts.

This **Chat API** documentation gives users and developers a clear understanding of how the **AI-powered chat feature** works in **TheDrive**, ensuring seamless interaction with files and AI models.