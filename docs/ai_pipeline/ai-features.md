
### docs/usage/ai-features.md

```markdown
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
const response = await fetch('/api/ai/query', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    query: "Summarize my project documents",
    context: "drive", // or "folder/:id", "file/:id"
    options: {
      include_citations: true,
      generate_visualization: false
    }
  })
});