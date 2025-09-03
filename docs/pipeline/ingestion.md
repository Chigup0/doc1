# How RAG Works for Multiple File Formats

## Universal File Ingestion Pipeline

**code snippet: Universal File Processing**

```python
def ingest_file(path: str, file_id: str, folder_id: Optional[str] = None, file_type: Optional[str] = None) -> int:
    """
    Universal file ingestion supporting PDF, TXT, DOCX, CSV.
    Automatically detects file type and applies appropriate processing.
    """

    # Detect file type if not provided
    if file_type is None:
        file_type = detect_file_type(path)

    print(f"[Ingest] Processing {file_type.upper()} file: {path}")

    # Load documents based on file type
    pdf_image_count = 0
    if file_type == 'pdf':
        # Check if PDF contains significant images
        if has_significant_images(path):
            print(f"[Ingest] PDF contains images - using hybrid processing")
            text_docs, pdf_image_count = process_pdf_with_images(path, file_id, folder_id)
            raw_docs = text_docs

            if pdf_image_count > 0:
                print(f"[Ingest] Processed {pdf_image_count} images from PDF")
        else:
            print(f"[Ingest] PDF has no significant images - using standard text processing")
            raw_docs = load_pdf(path)
    elif file_type == 'text':
        raw_docs = load_text(path)
    elif file_type == 'docx':
        raw_docs = load_docx(path)
    elif file_type == 'csv':
        raw_docs = load_csv(path)

    elif file_type in ["jpeg", "jpg", "png", "gif"]:
        print(f"[RAG] Ingesting image file: {path}")

        with open(path, "rb") as f:
            image_bytes = f.read()

        # Use vision model for image analysis
        llm_vision = ChatGoogleGenerativeAI(model="gemini-1.5-pro", google_api_key=settings.GEMINI_API_KEY)
        summary = get_image_summary(image_bytes, llm_vision)
        print(f"[RAG] Generated image summary: {summary[:100]}...")

        # Create a Document with the summary and metadata
        doc = Document(
            page_content=summary,
            metadata={
                "file_id": file_id,
                "folder_id": folder_id,
                "source": os.path.basename(path),
                "type": "image",
                "original_path": path
            }
        )

        # Add to the vector store
        vs = get_vs()
        vs.add_documents([doc])

        return 1
    else:
        print(f"[Ingest] Unsupported file type: {file_type}")
        return 0

    if file_type not in ["jpeg", "jpg", "png", "gif"]:
        if not raw_docs:
            print(f"[Ingest] No content extracted from {path}")
            return 0

        print(f"[Ingest] Extracted {len(raw_docs)} raw documents from {file_type.upper()}")

        # Chunk documents
        chunks = chunk_docs(raw_docs, file_type)
        print(f"[Ingest] Created {len(chunks)} chunks")
```

## File Type Specific Processing

1. **Ingestion:** Files are loaded and parsed according to their type (PDF, DOCX, CSV, images, etc.).
2. **Chunking:** Content is split into manageable, context-rich chunks.
3. **Embedding:** Chunks are embedded using LLM-powered models for semantic search.
4. **Graph Construction:** Entities and relationships are extracted and stored in Neo4j, enabling graph-based queries and cross-file knowledge.
5. **Retrieval:** When a user asks a question, relevant chunks are retrieved using semantic and graph-based search, and answers are generated with citations and context.
6. **Multimodal Analysis:**
   - **Image Preprocessing:** Uploaded images are enhanced (grayscale, contrast, sharpening, morphological operations) to improve OCR and visual analysis accuracy.
   - **OCR (Text Extraction):** Both Tesseract and EasyOCR are used to extract printed and handwritten text, with confidence filtering and bounding box localization. Results are deduplicated and combined for comprehensive coverage.
   - **Diagram Structure Analysis:** Diagrams are analyzed for geometric shapes (triangles, rectangles, circles, polygons) and lines/connections using edge detection and contour analysis. This enables understanding of visual relationships and flow.
   - **Chart and Graph Analysis:** Chart images are sent to a vision-capable LLM, which extracts chart type, axis labels, data points, trends, and key insights, making visual data explorable via chat.
   - **Entity Detection:** Images are analyzed by a vision LLM to identify entities (people, objects, text, structures, vehicles, animals, natural elements) with descriptions, locations, and attributes.
   - **Comprehensive Image Analysis:** All above methods are combined to produce a rich, structured summary of image content, including text, shapes, entities, and chart insights. Results are stored as searchable documents with metadata for downstream retrieval and chat-based exploration.

**code snippet: CSV Processing**

```python
def load_csv(path: str) -> List[Document]:
    """Load CSV files with intelligent processing."""
    try:
        # Try multiple encoding strategies
        csv_params = [
            {'encoding': 'utf-8'},
            {'encoding': 'latin-1'},
            {'encoding': 'cp1252'},
            {'encoding': 'utf-8', 'sep': ';'},
            {'encoding': 'latin-1', 'sep': ';'},
        ]

        df = None
        for params in csv_params:
            try:
                df = pd.read_csv(path, **params)
                break
            except:
                continue

        if df is None:
            print(f"[Ingest] Could not read CSV file {path}")
            return []

        docs = []

        # Strategy 1: Create overview document
        csv_text = f"CSV File: {os.path.basename(path)}\n"
        csv_text += f"Columns: {', '.join(df.columns.tolist())}\n"
        csv_text += f"Rows: {len(df)}\n\n"

        # Add column descriptions
        for col in df.columns:
            col_info = f"Column '{col}':\n"
            if df[col].dtype == 'object':
                unique_vals = df[col].dropna().unique()[:10]
                col_info += f"  Sample values: {', '.join(map(str, unique_vals))}\n"
            elif df[col].dtype in ['int64', 'float64']:
                col_info += f"  Range: {df[col].min()} to {df[col].max()}\n"
                col_info += f"  Mean: {df[col].mean():.2f}\n"

            csv_text += col_info + "\n"

        # Add sample rows
        csv_text += "Sample rows:\n"
        csv_text += df.head(5).to_string(index=False)

        docs.append(Document(
            page_content=csv_text,
            metadata={
                'source': os.path.basename(path),
                'file_type': 'csv',
                'loader': 'PandasCSV',
                'columns': df.columns.tolist(),
                'rows': len(df)
            }
        ))

        # Strategy 2: Create row-based documents for smaller CSVs
        if len(df) <= 1000:
            for idx, row in df.iterrows():
                row_text = f"Row {idx + 1} from {os.path.basename(path)}:\n"
                for col, val in row.items():
                    if pd.notna(val):
                        row_text += f"{col}: {val}\n"

                docs.append(Document(
                    page_content=row_text,
                    metadata={
                        'source': os.path.basename(path),
                        'file_type': 'csv',
                        'row_number': idx + 1,
                        'loader': 'PandasCSV'
                    }
                ))

        return docs

    except Exception as e:
        print(f"[Ingest] Error loading CSV {path}: {e}")
        return []
```
