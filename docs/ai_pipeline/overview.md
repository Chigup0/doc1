# AI-Powered Cross-Platform File Storage System

## Overview

This backend is designed to power a next-generation, secure, and intelligent file storage platform, inspired by services like Google Drive and OneDrive but enhanced with advanced AI capabilities. It supports both web and mobile clients, providing:

- **Multi-modal support:** Handles documents (PDF, DOCX, TXT, PPTX), spreadsheets (CSV, XLSX), and images (JPG, PNG), with robust parsing and indexing for each format.
- **AI-powered search and chat:** Enables users to ask natural language questions across all their files, with answers grounded in indexed content and supported by citations.
- **Knowledge graph construction:** Extracts entities and relationships from files using LLMs and stores them in a Neo4j graph database, enabling cross-file and cross-modal reasoning.
- **Multimodal analysis:** Processes images for text (OCR), diagrams, charts, and entities using Tesseract, EasyOCR, and vision models, making visual content searchable and explorable.
- **Privacy and security:** Implements user data isolation and access control, ensuring that AI and retrieval functions only operate on data the user is permitted to access.

The system is modular, extensible, and designed for competition-grade performance, with a focus on explainability, analytics, and user-centric privacy.

## Why Not Simple RAG?

Simple RAG systems retrieve text chunks and generate answers using LLMs, but they lack explicit modeling of entities, relationships, and provenance. This limits their ability to:

- **Aggregate knowledge across files and modalities:** Simple RAG cannot connect information from different sources or understand cross-file relationships.
- **Provide source-linked, explainable answers:** Without entity tracking, it's difficult to provide precise citations and explain reasoning paths.
- **Visualize knowledge connections:** Simple vector search doesn't capture the rich relationship structure that enables knowledge graph visualization.
- **Filter by confidence or validate extracted information:** Simple RAG lacks confidence scoring and validation mechanisms for extracted facts.
- **Support multi-hop reasoning and advanced analytics:** Vector similarity alone cannot support complex queries requiring multiple inference steps.

**GraphRAG** overcomes these limitations by:

1. **Building a knowledge graph from all files:** Entities and relations are extracted and linked, creating a structured knowledge representation.
2. **Supporting context-aware, cross-file, and cross-modal retrieval:** Graph queries can traverse relationships and combine vector similarity with structured knowledge.
3. **Providing source-linked answers:** Every fact is linked back to its source chunks, enabling precise citations and provenance tracking.
4. **Enabling advanced analytics:** Graph statistics, confidence metrics, and relationship analysis support deeper insights.
5. **Supporting explainable AI:** The graph structure makes reasoning paths visible and interpretable.

This approach is essential for the intelligent, secure, and multi-modal file storage system described above, where users need to explore complex relationships across diverse content types.
