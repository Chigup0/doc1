# File Operations API for TheDrive

This document provides a detailed overview of the **File Operations API** for **TheDrive**. It includes endpoints for **file uploads**, **folder creation**, **file management**, **AI processing**, and other file-related operations.

---

## 1. Endpoints Overview

The following endpoints are available in the File Operations API:

1. **POST /upload**: Uploads a file to **AWS S3**, stores metadata in the database, and optionally triggers AI processing.
2. **POST /folder**: Creates a new folder within a user's file system.
3. **PUT /item/{item_id}**: Renames an existing file or folder.
4. **DELETE /item/{item_id}**: Deletes a file or folder, including removal from **AWS S3** and related AI indexes.
5. **GET /items**: Retrieves a list of files and folders for a specific user and parent folder.
6. **GET /item/{item_id}/view-link**: Generates a pre-signed URL for viewing a file.
7. **GET /item/{item_id}/ai-status**: Retrieves the AI processing status for a file.

---

## 2. Endpoint Details

### 2.1 POST /upload

This endpoint allows users to upload files to the system. The file is stored in **AWS S3**, and metadata (such as file name, type, and S3 key) is saved in the **PostgreSQL** database. If **GraphRAG** (AI processing) is available, the file will be processed in the background.

#### Request Body:
- **parentId**: The ID of the parent folder (can be 'root' for the main folder).
- **file**: The file to be uploaded.

#### Response:
- **201 Created**: Successfully uploaded the file.

**Example Response**:
```
json
{
    "id": "file-12345",
    "name": "document.pdf",
    "type": "file",
    "s3_key": "user123/file-12345/document.pdf",
    "size_bytes": 102400,
    "mime_type": "application/pdf",
    "ai_processed": false,
    "ai_processing_status": "pending"
}
```
Example Code:
```
@router.post("/upload", response_model=schemas.FileSystemItem, status_code=201)
def upload_file(
    parentId: str,
    background_tasks: BackgroundTasks,
    file: UploadFile = File(...),
    db: Session = Depends(database.get_db),
    current_user: models.User = Depends(get_current_user)
):
    file_id = f"file-{uuid.uuid4()}"
    s3_key = f"{current_user.id}/{file_id}/{file.filename}"

    # Read file content into memory safely
    contents = file.file.read()
    file_size = len(contents)

    # Upload to S3
    s3_service.upload_file_obj(
        io.BytesIO(contents),
        settings.S3_BUCKET_NAME,
        s3_key,
        extra_args={
            "ContentType": file.content_type,
            "ContentDisposition": "inline",
        }
    )

    # Save metadata to DB
    new_file = models.FileSystemItem(
        id=file_id,
        name=file.filename,
        type="file",
        s3_key=s3_key,
        mime_type=file.content_type,
        size_bytes=file_size,
        owner_id=current_user.id,
        parent_id=None if parentId == 'root' else parentId,
        ai_processed=False,
        ai_processing_status="pending"
    )

    db.add(new_file)
    db.commit()
    db.refresh(new_file)

    # Process file for GraphRAG automatically
    if GRAPHRAG_AVAILABLE:
        background_tasks.add_task(
            process_file_for_graphrag,
            file_id=new_file.id,
            s3_key=s3_key,
            user_id=current_user.id,
            filename=file.filename
        )

    return new_file
```

2.2 POST /folder

This endpoint creates a new folder within a user's file system. The folder is saved in the database, and the parent folder (if any) is specified.
```
Request Body:
{
    "name": "New Folder",
    "parentId": "root"
}
```
Response:
```
201 Created: Successfully created the folder.
```
Example Response:
```
{
    "id": "folder-12345",
    "name": "New Folder",
    "type": "folder",
    "owner_id": 1,
    "parent_id": "root"
}
```

Example Code:
```
@router.post("/folder", response_model=schemas.FileSystemItem, status_code=201)
def create_folder(folder: schemas.FolderCreate, db: Session = Depends(database.get_db), current_user: models.User = Depends(get_current_user)):
    parent_db_id = None if folder.parentId == 'root' else folder.parentId
    
    # Check for duplicates
    existing = db.query(models.FileSystemItem).filter_by(
        owner_id=current_user.id, parent_id=parent_db_id, name=folder.name
    ).first()
    if existing:
        raise HTTPException(status_code=409, detail="An item with this name already exists")

    new_folder = models.FileSystemItem(
        id=f"folder-{uuid.uuid4()}", name=folder.name, type="folder",
        owner_id=current_user.id, parent_id=parent_db_id
    )
    db.add(new_folder)
    db.commit()
    db.refresh(new_folder)
    return new_folder
```

2.3 PUT /item/{item_id}

This endpoint allows users to rename a file or folder. It ensures that the new name doesn't conflict with other existing items in the same directory.

Request Body:
```
{
    "name": "Updated File Name"
}
```

Response:
```
200 OK: Successfully renamed the item.
```

Example Response:
```
{
    "id": "file-12345",
    "name": "Updated File Name",
    "type": "file",
    "s3_key": "user123/file-12345/document.pdf",
    "size_bytes": 102400,
    "mime_type": "application/pdf",
    "ai_processed": false,
    "ai_processing_status": "pending"
}
```

Example Code:
```
@router.put("/item/{item_id}", response_model=schemas.FileSystemItem)
def rename_item(item_id: str, item_update: schemas.ItemUpdate, db: Session = Depends(database.get_db), current_user: models.User = Depends(get_current_user)):
    item = db.query(models.FileSystemItem).filter_by(id=item_id, owner_id=current_user.id).first()
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    
    # Check for name conflicts
    existing = db.query(models.FileSystemItem).filter(
        models.FileSystemItem.id != item_id,
        models.FileSystemItem.parent_id == item.parent_id,
        models.FileSystemItem.name == item_update.name,
        models.FileSystemItem.owner_id == current_user.id
    ).first()
    if existing:
        raise HTTPException(status_code=409, detail="An item with this name already exists")

    item.name = item_update.name
    db.commit()
    db.refresh(item)
    return item
```

2.4 DELETE /item/{item_id}

This endpoint deletes a file or folder. If the item is a file, it will also be removed from AWS S3.

Response:
```
204 No Content: Successfully deleted the item.
```

Example Code:
```
@router.delete("/item/{item_id}", status_code=204)
def delete_item(item_id: str, db: Session = Depends(database.get_db), current_user: models.User = Depends(get_current_user)):
    item = db.query(models.FileSystemItem).filter_by(id=item_id, owner_id=current_user.id).first()
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    
    if item.type == 'file' and item.s3_key:
        s3_service.delete_file(settings.S3_BUCKET_NAME, item.s3_key)
        # Add background task to remove from AI indexes
    
    db.delete(item)
    db.commit()
```

2.5 GET /item/{item_id}/view-link

This endpoint generates a pre-signed URL for viewing a file directly from AWS S3. This URL can be used to access the file for a limited time.

Response:
```
200 OK: Returns the pre-signed URL for viewing the file.
```

Example Response:
```
{
    "url": "https://example.com/s3_presigned_url"
}
```

Example Code:
```
@router.get("/item/{item_id}/view-link", response_model=schemas.ViewLinkResponse)
def get_view_link(item_id: str, db: Session = Depends(database.get_db), current_user: models.User = Depends(get_current_user)):
    item = db.query(models.FileSystemItem).filter_by(id=item_id, owner_id=current_user.id).first()
    if not item or item.type != 'file' or not item.s3_key:
        raise HTTPException(status_code=404, detail="File not found or is not a viewable file.")
    
    url = s3_service.generate_presigned_url(settings.S3_BUCKET_NAME, item.s3_key)
    return {"url": url}
```

2.6 **GET /item/{item_id}/ai-status**

This endpoint allows users to retrieve the AI processing status of a file. The AI processing status includes information such as whether the file has been processed, the number of entities and relationships detected, and any processing errors.

Response:
```
200 OK: Returns the AI processing status of the file.
```

Example Response:
```
{
    "file_id": "file-12345",
    "processed": true,
    "status": "completed",
    "entities": 20,
    "relationships": 10,
    "communities": 5
}
```

Example Code:
```
@router.get("/item/{item_id}/ai-status")
def get_ai_status(item_id: str, db: Session = Depends(database.get_db), current_user: models.User = Depends(get_current_user)):
    """Get AI processing status for a file"""
    item = db.query(models.FileSystemItem).filter_by(id=item_id, owner_id=current_user.id).first()
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    
    return {
        "file_id": item_id,
        "processed": getattr(item, 'ai_processed', False),
        "status": getattr(item, 'ai_processing_status', 'pending'),
        "entities": getattr(item, 'ai_entities', 0),
        "relationships": getattr(item, 'ai_relationships', 0),
        "communities": getattr(item, 'ai_communities', 0)
    }
```

3. Conclusion

The File Operations API for TheDrive provides robust functionality for managing files and folders, including uploading, creating, renaming, deleting, and viewing files. Additionally, it supports AI processing for file contents and generates presigned URLs for secure file access. The integration with AWS S3 and PostgreSQL ensures efficient and scalable file storage and metadata management.

This API is designed to support users in managing their digital assets securely and intelligently, with the added benefit of AI-driven insights and graph-based search features.


---

### **Key Sections**:
1. **File Upload**: Handles file upload to **AWS S3** and metadata saving to **PostgreSQL**.
2. **Folder Creation**: Creates folders within the user’s file system.
3. **File Management**: Renaming and deletion of files, including removal from **S3**.
4. **AI Processing**: Includes endpoints for checking the AI status of files and generating pre-signed URLs for file access.