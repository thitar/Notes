---
title: Setting Up an Archiving Workflow in Paperless-ngx
updated: 2025-07-22 20:47:15Z
created: 2025-07-22 20:46:34Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

## Setting Up an Archiving Workflow in Paperless-ngx

The goal is to create a seamless process from physical paper to a searchable, organized digital archive.

### Prerequisites:

- **Paperless-ngx LXC:** You already have this set up. Ensure it has sufficient CPU, RAM (especially for OCR), and storage.
    
- **Storage for Paperless-ngx:** The LXC needs access to persistent storage for the `data`, `media`, and `consume` directories. For a Proxmox LXC, this usually means mounting a bind mount or a dedicated mount point from your Proxmox host.
    
    - `PAPERLESS_CONSUMPTION_DIR`: Where new documents land.
        
    - `PAPERLESS_MEDIA_ROOT`: Where Paperless-ngx stores the processed documents.
        
    - `PAPERLESS_DATA_DIR`: For the database and search index.
        
- **Scanner:** A scanner capable of scanning to a network share (SMB/NFS) or email is highly recommended for automating input into the `consume` folder.
    

### Workflow Steps & Setup:

Here’s a common and efficient archiving workflow:

#### Step 1: Inputting Documents into the `consume` Folder

This is where your physical (or digital) documents first enter the system.

- **Option A: Network Scanner (Recommended for Physical Docs)**
    
    1.  **Configure a Shared Folder:** On your Proxmox host (or a separate file server), create a Samba/NFS share that points to your Paperless-ngx `consume` directory (the host path that the LXC maps to `PAPERLESS_CONSUMPTION_DIR`).
        
    2.  **Scanner Configuration:** Configure your network scanner to scan directly to this shared folder.
        
    3.  **Process:** Scan your physical documents. They will appear in the `consume` folder, and Paperless-ngx will automatically pick them up.
        
    
    - **Tip:** Some scanners can create multi-page PDFs, which is ideal.
- **Option B: Email Integration (Good for Digital Docs from Email)**
    
    1.  **Configure Mail Accounts:** In Paperless-ngx (Admin > Mail Accounts), set up your email accounts (e.g., a dedicated email address like `scan@yourdomain.com`).
        
    2.  **Create Mail Rules:** (Admin > Mail Rules) Define rules to process incoming emails. For example:
        
        - **Trigger:** "Any incoming mail"
            
        - **Conditions:** "Sender is `utility_company@example.com`"
            
        - **Actions:** "Assign Correspondent: Utility Company," "Assign Document Type: Invoice," "Assign Tag: Bills."
            
    
    - Paperless-ngx can also parse attachments.
- **Option C: Manual Upload (For one-off digital files)**
    
    1.  Simply drag and drop files directly into the Paperless-ngx web interface.
        
    2.  You can also place files manually into the `consume` folder via your file manager if you have direct access.
        

#### Step 2: Automatic Consumption & OCR

Paperless-ngx continuously monitors the `consume` folder.

- **Ensure Consumer is Running:** In your LXC, the Paperless-ngx consumer process should be running. If using Docker Compose (common for LXC installs), this is usually handled automatically.
    
- **OCR Languages:** Go to Paperless-ngx settings (Admin > OCR). Ensure the correct OCR languages are enabled for your documents. This is crucial for accurate text extraction.
    

#### Step 3: Metadata Assignment (Automated & Manual)

This is where documents get their structure.

- **Automatic Matching:**
    
    - As you categorize documents, Paperless-ngx learns patterns.
        
    - When a new document is consumed, it will attempt to assign a Correspondent, Document Type, and Tags based on its content and your historical data.
        
    - The more consistently you apply metadata, the better the auto-matching becomes.
        
- **Manual Review and Correction:**
    
    - After consumption, documents appear in your dashboard.
        
    - **Review Regularly:** Periodically check new documents. Paperless-ngx will often suggest categories or mark documents as "needs review" if it's uncertain.
        
    - **Edit Document:** Click on a document to open its details page. Here, you can:
        
        - **Assign/Correct Correspondent:** Select the correct person/company.
            
        - **Assign/Correct Document Type:** Select the type (e.g., Invoice, Statement).
            
        - **Add Tags:** This is where you get granular. Add as many relevant tags as needed (e.g., "Electricity," "Q1 2025," "Household").
            
        - **Set Title & Document Date:** Adjust if needed. The document date is usually autodetected.
            

#### Step 4: Structuring Files with Storage Paths (Optional but Recommended)

If you want your actual files to be organized on disk in a human-readable way (e.g., for external backup or direct access), configure `PAPERLESS_FILENAME_FORMAT`.

1.  **Edit `docker-compose.env` (or `paperless.conf` if not using Docker):** Add or modify the `PAPERLESS_FILENAME_FORMAT` variable.
    
    - A good starting point: `PAPERLESS_FILENAME_FORMAT="{created_year}/{correspondent}/{document_type}/{title}"`
        
    - This would create a structure like: `2025/My Bank/Bank Statement/Monthly Statement Jan 2025.pdf`
        
    - You can also include `"{asn}"` (Archive Serial Number) if you use physical numbering.
        
2.  **Define Storage Paths (Admin > Storage Paths):** You can create named storage paths to organize your documents based on rules. For example, you might create a storage path called "Taxes" and use a workflow to assign documents tagged "Taxes" to this path. This allows the physical file to be moved into a specific folder on your media storage.
    

#### Step 5: Automating with Workflows

Workflows are powerful for more complex automation beyond simple auto-matching.

1.  **Go to Admin > Workflows.**
    
2.  **Create New Workflow:**
    
    - **Trigger:** Choose when the workflow should run (e.g., "Document consumed," "Document updated").
        
    - **Conditions:** Define criteria for the document (e.g., "Has any of tags: 'Taxes'", "Correspondent is: 'Utility Company'", "Full text contains: 'invoice number'").
        
    - **Actions:** Define what happens if conditions are met:
        
        - `Assign tag`: Automatically add specific tags.
            
        - `Assign correspondent`: Set the correspondent.
            
        - `Assign document type`: Set the document type.
            
        - `Assign storage path`: Move the physical file to a specific path on disk.
            
        - `Assign owner/permissions`: Control who can view/edit the document.
            
        - `Set Custom Field`: If you use custom metadata fields.
            
        - `Run Script`: For highly custom actions (e.g., sending a notification, interacting with another API).
            
    
    **Example Workflow (for an archiving scenario):**
    
    - **Name:** "Process Tax Documents"
        
    - **Trigger:** "Document consumed"
        
    - **Conditions:**
        
        - "Full text contains 'tax return' OR 'IRS' OR 'steuererklärung'"
            
        - "Has any of tags: 'Financial'" (if you have a broader tag)
            
    - **Actions:**
        
        - "Assign tag: 'Taxes', '{created_year} Taxes'" (e.g., "2025 Taxes")
            
        - "Assign storage path: 'Taxes Archive'" (assuming you defined a storage path that corresponds to `media/documents/Taxes Archive`)
            
        - "Assign owner: your_user_account"
            
        - "Grant view permissions to group: Accountants" (if applicable)
            

#### Step 6: Backup Strategy

- **Regular Backups:** Crucial for any document management system.
    
- **Backup Components:**
    
    - **`data` directory:** Contains your database and search index.
        
    - **`media` directory:** Contains all your actual processed documents.
        
    - **Database Dump:** Perform regular database dumps (e.g., using `pg_dump` for PostgreSQL or `mysqldump` for MariaDB).
        
- **Paperless-ngx Exporter:** Paperless-ngx also has a built-in document exporter that can export your documents along with their metadata in a structured format, making it easier to migrate or recover.