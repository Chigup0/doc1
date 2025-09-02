# TheDrive - System Architecture Overview

**TheDrive** is an **AI-powered cloud storage system** that integrates **secure file storage**, **AI-driven content analysis**, and **intelligent querying** for users. The system is designed with a **microservices architecture** that ensures scalability, security, and flexibility.

---

## 1. High-Level Architecture

TheDrive’s architecture is divided into several key components:

### 1.1 Frontend (Next.js)
- **Role**: Provides a responsive user interface for file management, querying, and interacting with AI-powered features.
- **Technologies**: **Next.js** for server-side rendering and static page generation, **Tailwind CSS** for styling, and **Axios** for API requests.

### 1.2 Backend (FastAPI)
- **Role**: Handles file management, user authentication, and interactions with AI models. The backend serves as the main API layer for the system.
- **Technologies**: **FastAPI** for building the RESTful API, **SQLAlchemy** for database interaction, and **JWT** for authentication.

### 1.3 Cloud Storage (AWS S3)
- **Role**: Stores files securely in the cloud with high scalability and availability.
- **Technologies**: **AWS S3** for scalable file storage, leveraging **IAM** for secure access.

### 1.4 Database (PostgreSQL & Neo4j)
- **PostgreSQL**: Used for storing user credentials, file metadata, and other structured data.
- **Neo4j**: A graph database that models relationships between files and metadata, enabling AI-based queries and semantic search.

### 1.5 AI Layer
- **Role**: Processes files for AI-driven insights, such as text extraction, entity recognition, and semantic search.
- **Technologies**: **GraphRAG** for processing documents, **PyTorch**/**TensorFlow** for machine learning models, and **OCR** for image processing.

---

## 2. Security and Privacy

- **JWT Authentication**: Used for user authentication and maintaining secure sessions.
- **End-to-End Encryption (E2EE)**: All user data is encrypted at rest and in transit (future enhancement, currently in progress).
- **Data Segregation**: Each user's data is isolated to ensure privacy and security.

---

## 3. Data Flow

1. **File Upload**: Files are uploaded via the frontend, stored in **AWS S3**, and metadata is saved in **PostgreSQL**.
2. **AI Processing**: Once a file is uploaded, it is processed by the AI layer to extract insights (e.g., entities, relationships).
3. **User Queries**: Users can query files via the frontend, with AI-driven results fetched from **PostgreSQL** and **Neo4j**.

---

## Conclusion

TheDrive combines **secure cloud storage**, **AI-powered processing**, and **advanced querying capabilities** to offer users an intelligent and scalable solution for managing their digital assets. The system is built with a modular architecture that ensures flexibility, security, and high performance.
