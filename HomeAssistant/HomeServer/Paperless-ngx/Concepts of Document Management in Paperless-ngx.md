---
title: Concepts of Document Management in Paperless-ngx
updated: 2025-07-22 20:46:25Z
created: 2025-07-22 20:44:46Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

## Concepts of Document Management in Paperless-ngx

Paperless-ngx is built around the idea of making your documents searchable and easily retrievable without relying on traditional folder structures. It achieves this through several key concepts:

1.  **Consumption Directory (Consume Folder):**
    
    - This is the "inbox" for Paperless-ngx. You place your digital documents (scans, PDFs, images, etc.) into this designated directory.
        
    - Paperless-ngx actively monitors this folder. When a new document appears, it automatically "consumes" it.
        
    - During consumption, the document is processed:
        
        - **OCR (Optical Character Recognition):** If the document is an image-based PDF or an image, Paperless-ngx uses OCR to extract searchable text from it. This is a game-changer for scanned documents.
            
        - **Automatic Archiving:** The processed document is then moved out of the `consume` folder and into Paperless-ngx's internal storage (`media` directory), often in a PDF/A format for long-term preservation.
            
        - **Metadata Extraction:** Paperless-ngx tries to automatically detect and assign metadata like date, correspondent, and document type using its built-in intelligence and learning from your past categorizations.
            
2.  **Documents:**
    
    - Once consumed, a document becomes an entry in Paperless-ngx. It's not just the file; it's the file *plus* all its associated metadata.
        
    - Paperless-ngx keeps the original document and, if OCR was performed, a searchable PDF/A version.
        
3.  **Correspondents:**
    
    - A correspondent is typically the **person, institution, or company** that a document originates from or is sent to. Think of it as "who sent this" or "who is this about."
        
    - Examples: "My Bank," "Utility Company," "Doctor's Office," "Employer."
        
    - Paperless-ngx can learn to automatically assign correspondents based on the document's content.
        
4.  **Document Types:**
    
    - A document type defines the **nature or category** of the document.
        
    - Examples: "Invoice," "Bank Statement," "Contract," "Receipt," "Letter," "Warranty."
        
    - Similar to correspondents, Paperless-ngx can learn to suggest or automatically assign document types.
        
5.  **Tags:**
    
    - Tags are highly flexible labels that you can assign to documents. Unlike folders, a single document can have **multiple tags**. This is one of Paperless-ngx's most powerful features for organization.
        
    - Think of them as keywords or topics.
        
    - Examples: "Taxes 2024," "Car Maintenance," "Home Improvement," "Medical," "Insurance."
        
    - You can use tags to cross-reference documents across different correspondents or document types. For instance, an "Invoice" from "My Bank" might also be tagged "Loan" and "2025."
        
6.  **Storage Paths:**
    
    - While Paperless-ngx primarily uses its internal database and a flat file structure by default, you can configure storage paths to organize your *physical* files in a more human-readable folder structure *after* consumption. This is useful if you also want to access documents outside of Paperless-ngx or for external backups that maintain a logical hierarchy.
        
    - You can use placeholders (e.g., `{correspondent}/{document_type}/{title}`) to automatically generate folder paths and filenames.
        
7.  **Workflows:**
    
    - Workflows allow you to automate actions based on certain triggers and conditions. They provide a powerful way to streamline your document processing.
        
    - Triggers can include "Document consumed," "Document updated," or "Email received."
        
    - Actions can include assigning correspondents, document types, tags, storage paths, permissions, or even executing custom scripts.
        
8.  **Automatic Matching:**
    
    - Paperless-ngx uses machine learning to "learn" from your categorizations. The more you assign correspondents, document types, and tags to documents, the better it becomes at suggesting or automatically applying them to new, similar documents. This significantly speeds up the archiving process over time.
