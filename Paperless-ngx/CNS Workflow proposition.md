---
title: CNS Workflow proposition
updated: 2025-07-21 22:30:32Z
created: 2025-07-21 22:28:13Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

Excellent question. This is a perfect, real-world use case for Paperless-ngx, and everything you've described is absolutely achievable. Your setup (Proxmox LXC, mail import) is the ideal foundation.

Let's break this down into the general workflow and then the specific, detailed workflow for your medical bills.

### Part 1: The General "Scan-to-Archive" Workflow

This is the basic process for getting any document into Paperless-ngx.

1.  **On the Samsung Printer:**
    *   Configure the "Scan to Email" feature.
    *   Set the destination email address to the one your Paperless-ngx instance is monitoring.
    *   **Pro-Tip:** Choose PDF as the output format. It's ideal for OCR. If you can, set the resolution to 300 DPI, which is the sweet spot for quality and file size for OCR.

2.  **On the Synology NAS (Optional but Recommended):**
    *   Your Paperless-ngx LXC is likely using storage on your Proxmox host. For long-term safety, you should have a backup plan.
    *   Configure a backup job in Proxmox to regularly back up the entire Paperless-ngx LXC to your Synology NAS. This protects your documents, database, and configuration.

3.  **Inside Paperless-ngx (The Magic Happens Here):**
    *   **Mail Import:** Paperless-ngx checks the configured email account, finds the email from your printer, and downloads the attached PDF.
    *   **Consumption:** The PDF is moved to the `consume` directory. Paperless-ngx automatically picks it up.
    *   **OCR (Optical Character Recognition):** This is the key step. Paperless will process the PDF and create a text layer. To handle all your languages, you must ensure you have configured them correctly. In your Paperless-ngx configuration (e.g., `paperless.conf` or environment variables), you need this setting:
        ```
        PAPERLESS_OCR_LANGUAGES=eng deu fra
        ```
        This tells the OCR engine (Tesseract) to use the language packs for English, German, and French, allowing it to accurately recognize text in any of those languages on the same document.
    *   **Indexing:** The extracted text and metadata are added to a searchable index.
    *   **The Inbox:** The newly processed document lands in your Paperless-ngx "Inbox" view, waiting for your review. This is where your manual workflow begins. You'll tag it, assign a correspondent, a document type, and a date. Much of this can be automated over time as Paperless learns from your actions.

---

### Part 2: The Detailed Medical Bill Workflow

This is the core of your request. It involves states, actions, and linking related documents. We will use a combination of **Tags**, **Custom Fields**, and **internal document links**.

#### **Initial Setup in Paperless-ngx**

Before you start, create the necessary building blocks in Paperless-ngx:

1.  **Create Tags:** Go to `Settings -> Tags`. Create the following tags:
    *   `medical`
    *   `bill`
    *   `status:unpaid` (Using a prefix like `status:` helps group them)
    *   `status:paid`
    *   `status:cns-submitted`
    *   `status:axa-submitted`

2.  **Create Document Types:** Go to `Settings -> Document Types`. Create:
    *   `Medical Bill`
    *   `CNS Report`
    *   `AXA Report`

3.  **Create Correspondents:** Go to `Settings -> Correspondents`. Create:
    *   `CNS` (Caisse Nationale de Santé)
    *   `AXA`
    *   The various doctors/clinics that send you bills.

4.  **Create Custom Fields (CRITICAL for Linking):** This is the most robust way to link reports. Go to `Settings -> Custom Fields`.
    *   **Name:** `Related CNS Report`
    *   **Type:** `Document Link`
    *   **Name:** `Related AXA Report`
    *   **Type:** `Document Link`
    *   **Name:** `Related Bills`
    *   **Type:** `Document Link` (and check the "Allow multiple documents" box)

---

#### The Step-by-Step Workflow in Action

**Step A: Scan the Initial Medical Bill**

1.  You receive a bill from your doctor.
2.  Scan it using your Samsung printer, which emails it to Paperless-ngx.
3.  The document appears in your Paperless Inbox.
4.  Process it:
    *   **Correspondent:** The Doctor/Clinic
    *   **Document Type:** `Medical Bill`
    *   **Tags:** `medical`, `bill`, `status:unpaid`
    *   **Title:** Give it a clear title, e.g., "Dr. Muller - Consultation 2023-10-26"
    *   Save it.

**Step B: Pay the Bill and Submit to CNS**

1.  You pay the bill via your bank.
2.  Find the bill in Paperless-ngx. A great way to do this is to have a "Saved View" for all documents with the `status:unpaid` tag.
3.  Edit the document:
    *   Remove the `status:unpaid` tag.
    *   Add the `status:paid` and `status:cns-submitted` tags.
4.  You now physically send this bill (or a group of them) to the CNS.

**Step C: Receive and Link the CNS Report**

1.  CNS processes your bills and sends you back a compilation report (décompte).
2.  Scan this CNS report into Paperless-ngx.
3.  Process the new document in your Inbox:
    *   **Correspondent:** `CNS`
    *   **Document Type:** `CNS Report`
    *   **Title:** e.g., "CNS Décompte 2023-11-15"
    *   **Tags:** `medical`
4.  **This is the linking step:**
    *   Edit the new CNS Report document.
    *   Go to the **Custom Fields** section.
    *   In the **`Related Bills`** field, start typing the title (or document ID) of the individual bills that this report covers. Paperless-ngx will let you search and select them. You can add multiple bills here.
    *   Save the CNS Report.

*Now, when you view the CNS report, you will have direct links to every individual bill it pertains to!*

**Step D: Submit to AXA and Link the AXA Report**

1.  You send the CNS report to your complementary insurance, AXA.
2.  Find the CNS Report in Paperless. Edit it and add the tag `status:axa-submitted`.
3.  AXA sends you their own reimbursement report.
4.  Scan the AXA report into Paperless-ngx.
5.  Process the new document in your Inbox:
    *   **Correspondent:** `AXA`
    *   **Document Type:** `AXA Report`
    *   **Title:** e.g., "AXA Reimbursement 2023-12-01"
    *   **Tags:** `medical`
6.  **Link it back:**
    *   Edit the new AXA Report document.
    *   Go to the Custom Fields section.
    *   In the **`Related CNS Report`** field, search for and select the CNS report you sent them.
    *   Save the AXA Report.

### Summary of the Final State

*   **An individual Medical Bill** will have tags like `status:paid` and `status:cns-submitted`.
*   **A CNS Report** will have a custom field `Related Bills` pointing to all the original medical bills. It might have a tag like `status:axa-submitted`.
*   **An AXA Report** will have a custom field `Related CNS Report` pointing to the CNS report.

This creates a clear, clickable, and logical hierarchy. When you open the AXA report, you can click the link to the CNS report. When you open the CNS report, you can see and click on all the individual bills that were part of that submission. This is an incredibly powerful way to keep your records organized and auditable.