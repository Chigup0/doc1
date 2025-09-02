# Data Flow in TheDrive

This document provides a detailed explanation of the **data flow** in **TheDrive**, including how data moves through the system from the frontend to the backend, storage, AI processing, and how user queries are handled.

---

## 1. High-Level Data Flow

TheDrive follows a **user-centered** flow where data is processed and managed through several stages:

1. **User Uploads File**:
   - The user uploads a file via the frontend interface.
   - The file is processed, stored, and metadata is stored in the backend.
  
2. **Metadata Storage**:
   - File metadata (e.g., file name, type, size) is saved to **PostgreSQL**.
   - The file is uploaded to **AWS S3** and its **S3 key** is saved in the database.
   
3. **AI Processing**:
   - Once the file is uploaded, the system triggers **AI processing** to analyze the contents (e.g., text, images).
   - Metadata related to AI analysis (e.g., **entities**, **relationships**, **communities**) is stored in the database.

4. **User Queries Files**:
   - The user interacts with the system by querying files via the frontend interface.
   - The backend processes the query by referencing the file metadata and AI-driven insights (e.g., semantic search, AI-based content retrieval).

---

## 2. Detailed Data Flow

### 2.1 File Upload Process

#### User Action:
- The user selects a file to upload through the **Next.js frontend**.
- The file is sent via an **HTTP POST request** to the backend API (FastAPI).

#### Backend Action:
- **FastAPI** receives the file and triggers the **AWS S3** upload process.
- The file is stored in an **S3 bucket**. S3 generates a **unique storage key** for the file, which is used for future access and reference.

#### Metadata Storage:
- Once the file is uploaded to **S3**, **metadata** (such as **file name**, **file type**, **file size**, and **S3 key**) is stored in the **PostgreSQL database**.
- A new entry is created in the **FileSystemItem** table, which references the uploaded file.

**Example Data Flow**:
1. User uploads file via frontend.
2. File is sent to FastAPI backend.
3. FastAPI stores the file in **AWS S3**.
4. File metadata is inserted into **PostgreSQL**.

```
python
# FastAPI backend receives the file
def upload_file(file: UploadFile = File(...), db: Session = Depends(get_db)):
    s3_key = upload_to_s3(file)  # Upload file to AWS S3
    file_metadata = {
        'name': file.filename,
        'type': 'file',  # or 'folder'
        's3_key': s3_key,
        'size_bytes': len(file.file.read())
    }
    # Store metadata in PostgreSQL
    new_file = models.FileSystemItem(**file_metadata)
    db.add(new_file)
    db.commit()
    return {"message": "File uploaded successfully!"}
```