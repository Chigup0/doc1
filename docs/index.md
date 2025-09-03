# TheDrive - AI-Powered Cross-Platform File Storage System

## Project Overview

**TheDrive** is an **AI-powered cloud storage platform** designed to transform the way users store, interact with, and manage their digital assets. Unlike traditional cloud storage systems like Google Drive or OneDrive, TheDrive integrates **advanced AI technologies** to provide smarter, more intuitive ways to interact with files, derive meaningful insights, and enhance overall user experience. 

With the growing concerns around **data privacy** and the increasing need for **security**, TheDrive has been built to not only offer intelligent features but also ensure that user data remains **safe, secure, and private**. This platform is not just about storing files—it's about empowering users with the tools to **analyze**, **manage**, and **gain insights** from their digital assets, all while maintaining the highest standards of security.

## Problem Statement

As the amount of digital data grows, users are faced with challenges not only in managing this information but also in ensuring its security and privacy. Traditional cloud storage systems allow for basic file storage, sharing, and collaboration, but they fall short when it comes to the following:

- **Intelligence**: Current systems do not provide advanced tools to **analyze** and **interact** with file contents effectively.
- **Security**: While they offer security features, many systems are vulnerable to data breaches or unauthorized access, often exposing user data to administrators and third parties.
- **User Experience**: Current solutions don't integrate **AI-powered features** such as **content-based search**, **query resolution**, or **semantic understanding** of data in a seamless and intuitive way.

TheDrive aims to solve these issues by creating an intelligent, secure cloud storage system that enables users to interact with their files in ways never before possible.

## Key Features

### 1. Multi-File Format Support

**TheDrive** supports a wide range of file formats, making it a versatile platform for various use cases:

- **Document Formats**: `.pdf`, `.docx`, `.txt`, `.pptx` (PowerPoint presentations)
- **Spreadsheet Formats**: `.csv`, `.xlsx` (Excel spreadsheets)
- **Image Formats**: `.jpg`, `.png`

Each of these file types is processed and displayed with **content integrity**, preserving the structure, layout, and formatting. This ensures that no matter the file type, users can interact with their files in a seamless, efficient way.

### 2. AI-Driven Smarter Storage System

#### a. Contextual Chat

One of the most innovative features of **TheDrive** is its **AI-powered contextual chat system**. This allows users to interact with their storage by asking questions about their files, folders, or documents, and the system will provide relevant answers. 

- **Chat Over Whole Drive**: 
   - Users can ask questions about the entire contents of their drive, such as "What documents contain the word 'budget'?" or "Show me the documents that mention 'project proposal'." The system can also calculate **aggregation metrics**, such as the total number of words across all documents.
   - This feature goes beyond just keyword search—it's about understanding the **context** of the files, providing more intelligent responses.

- **Chat Over Folders**:
   - Users can query specific folders without revealing information from other folders, ensuring **data privacy** at the folder level. For example, a user could ask, "How many documents are there in this folder related to 'marketing'?" and receive a relevant answer without exposing files from other folders.

- **Chat Over Documents**:
   - Users can ask about specific documents, and the AI will provide context and insights based on the content of the file.

#### b. Highlight-Based Query Resolution and Explanation

**TheDrive** goes beyond just storing files—users can **highlight specific text** or portions of a document, image, or table, and the system will provide **AI-powered insights**. 

- **Text Highlighting**: 
   - When a user highlights a word or phrase, the system will provide its **definition**, **usage**, and related explanations. This makes it easy for users to quickly understand complex terms without leaving the platform.
   
- **Image/Picture Interaction**:
   - For images or presentations, users can highlight portions of a visual (e.g., a chart, diagram, or handwritten note) to get an AI-generated description or analysis of that section.

- **Table Analysis**:
   - In spreadsheet files, users can select portions of tables to get **aggregation metrics**, **summaries**, or insights based on the data, helping users analyze their files more efficiently.

#### c. Cross-Referencing Files and Sections

**Cross-referencing** is a key feature that allows users to trace the origins of the data used to answer a query. Whenever a user asks a question about a document, the system provides:

- **Source Citation**: Links to the specific file and section that answered the query.
- **Chunk Highlighting**: When opening the linked file, the exact section of text or data used for the answer will be highlighted for quick reference.

#### d. Other AI-Powered Features

- **Image Understanding**: 
   - TheDrive can analyze and understand content within images, such as handwritten notes (via OCR), charts, diagrams, and more. Users can ask the system to **describe**, **explain**, or **analyze** parts of images, making the system more versatile for users dealing with visual data.
   
- **Semantic Search**: 
   - Unlike traditional search engines that rely solely on keywords, **semantic search** allows users to search based on the **meaning** and **context** of the query, returning more relevant and accurate results.

---

## System Architecture

TheDrive is built using a **microservices architecture** with a **serverless backend** and **scalable cloud infrastructure** to ensure both reliability and performance.

### Key Components:
- **Frontend** (Next.js):
   - Provides a dynamic, responsive web application that enables users to interact with their files, perform searches, and access AI-powered insights.
   
- **Backend** (FastAPI):
   - Powers the core services of the application, handling user authentication, file management, and AI query processing. It communicates with databases and AI models to provide intelligent responses.
   
- **AI Layer**:
   - A RAG+KAG pipeline that process text, images, and tables to provide users with insights, document summaries, and analytics.

- **Database Layer**:
   - The system uses a combination of **SQL databases** (for general metadata and user data) and **graph databases** (Neo4j) for representing complex relationships between files and metadata.

- **Cloud Integration**:
   - The system is hosted on **cloud platforms** like AWS and Azure, ensuring scalability, high availability, and reliability.

### Security
- **Data Isolation**: Each user’s data is physically and logically isolated to prevent unauthorized access or leakage.

---


