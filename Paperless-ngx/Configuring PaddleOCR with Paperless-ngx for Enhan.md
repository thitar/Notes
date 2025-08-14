---
title: Configuring PaddleOCR with Paperless-ngx for Enhanced OCR Quality
updated: 2025-08-01 22:19:26Z
created: 2025-08-01 22:19:13Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Configuring PaddleOCR with Paperless-ngx for Enhanced OCR Quality

This guide provides detailed instructions to set up a PaddleOCR service on your 'docker' VM and integrate it with Paperless-ngx using a `PAPERLESS_PRE_CONSUME_SCRIPT`. This ensures all your documents are OCR'd with PaddleOCR's advanced capabilities, leveraging your Intel N100's iGPU.

**Prerequisites:**

- Your 'docker' VM is running and has Docker installed.
    
- Your Intel N100's iGPU is successfully passed through to your 'docker' VM (as per Guide 1, Step 2).
    
- Paperless-ngx is installed and running on your 'docker' VM (as per Guide 1, Steps 3 & 4).
    
- You have SSH access to your 'docker' VM.
    

## Step 1: Deploy PaddleOCR as a Service on Your 'docker' VM

We will create a FastAPI web service that encapsulates PaddleOCR. This service will receive a PDF, OCR it, and return a new searchable PDF.

1.  **Create a dedicated directory** for the PaddleOCR service on your 'docker' VM:
    
    Bash
    
    ```
    mkdir ~/paddleocr-service
    cd ~/paddleocr-service
    ```
    
2.  Create app.py (FastAPI application):
    
    This Python script will handle PDF-to-image conversion, PaddleOCR, and then re-embedding the text into a new searchable PDF.
    
    Python
    
    ```
    # app.py
    from fastapi import FastAPI, UploadFile, File, HTTPException
    from fastapi.responses import StreamingResponse
    from paddleocr import PaddleOCR
    import io
    import numpy as np
    from PIL import Image
    import fitz # PyMuPDF
    from reportlab.pdfgen import canvas
    from reportlab.lib.units import inch
    from PyPDF2 import PdfWriter, PdfReader
    import tempfile
    import os
    import logging
    
    logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
    logger = logging.getLogger(__name__)
    
    app = FastAPI()
    
    # Initialize PaddleOCR engine.
    # enable_hpi=True attempts to use high-performance inference (e.g., OpenVINO for Intel iGPU).
    # This will download models on the first run.
    ocr_engine = PaddleOCR(use_angle_cls=True, lang='en', enable_hpi=True) # Adjust 'lang' as needed, e.g., 'ch' for Chinese [4]
    
    @app.post("/ocr-pdf-searchable/")
    async def ocr_pdf_searchable(file: UploadFile = File(...)):
        if not file.content_type == "application/pdf":
            raise HTTPException(status_code=400, detail="Only PDF files are supported.")
    
        logger.info(f"Received PDF for OCR: {file.filename}")
    
        try:
            pdf_bytes = await file.read()
    
            # Create a temporary file for the input PDF
            with tempfile.NamedTemporaryFile(delete=False, suffix=".pdf") as temp_input_pdf:
                temp_input_pdf.write(pdf_bytes)
                temp_input_pdf_path = temp_input_pdf.name
    
            doc = fitz.open(temp_input_pdf_path)
            pdf_writer = PdfWriter()
    
            for page_num in range(len(doc)):
                page = doc.load_page(page_num)
                # Render page to image at 300 DPI for optimal OCR quality
                pix = page.get_pixmap(matrix=fitz.Matrix(300/72, 300/72))
                img_pil = Image.fromarray(np.frombuffer(pix.samples, dtype=np.uint8).reshape(pix.h, pix.w, pix.n))
    
                # Perform OCR on the image
                # result is a list of lines, each line is [bbox, (text, confidence)]
                result = ocr_engine.ocr(img_pil, det=True, rec=True, cls=True)
    
                # Create a new PDF page with the original image and an invisible text layer
                packet = io.BytesIO()
                # Calculate page dimensions in points (1/72 inch)
                page_width_pt = pix.width * (72/300)
                page_height_pt = pix.height * (72/300)
                can = canvas.Canvas(packet, pagesize=(page_width_pt, page_height_pt))
    
                # Draw the original image onto the canvas
                img_temp_path = None
                with tempfile.NamedTemporaryFile(delete=False, suffix=".png") as temp_img_file:
                    img_temp_path = temp_img_file.name
                    img_pil.save(img_temp_path, format='PNG')
                can.drawImage(img_temp_path, 0, 0, width=page_width_pt, height=page_height_pt)
                os.unlink(img_temp_path) # Clean up temp image file
    
                # Add invisible text layer
                can.setFillColorRGB(0,0,0, alpha=0) # Make text invisible
                can.setFont("Helvetica", 1) # Small base font size, will be scaled
    
                if result and result: # Ensure result is not empty and has at least one page
                    for line in result: # Iterate over lines in the first (and only) image processed
                        bbox_coords = line # [[x1,y1],[x2,y2],[x3,y3],[x4,y4]]
                        text = line[1] # The recognized text
    
                        # Convert PaddleOCR's 4-point bbox to (x, y, width, height) for PDF
                        x_coords = [p for p in bbox_coords]
                        y_coords = [p[1] for p in bbox_coords]
                        min_x, max_x = min(x_coords), max(x_coords)
                        min_y, max_y = min(y_coords), max(y_coords)
                        width_px = max_x - min_x
                        height_px = max_y - min_y
    
                        # Scale coordinates from image pixels (300 DPI) to PDF points (72 DPI)
                        scale_factor = 72 / 300
                        x_pt = min_x * scale_factor
                        y_pt = (pix.height - max_y) * scale_factor # PDF y-axis is inverted
    
                        # Draw text as a text object to ensure it scales with bounding box
                        textobject = can.beginText()
                        textobject.setTextOrigin(x_pt, y_pt)
                        # Adjust font size to fit the height of the bounding box
                        # A factor like 0.8-0.9 helps prevent text from overflowing
                        font_size = height_px * scale_factor * 0.8
                        if font_size > 0: # Avoid zero or negative font sizes
                            can.setFont("Helvetica", font_size)
                            can.drawString(x_pt, y_pt, text)
                        textobject.textLine(text)
                        can.drawText(textobject)
                can.save()
    
                # Add the newly created PDF page to the PdfWriter
                packet.seek(0)
                new_pdf_page = PdfReader(packet).pages
                pdf_writer.add_page(new_pdf_page)
    
            # Save the final searchable PDF to a BytesIO object
            output_pdf_bytes_io = io.BytesIO()
            pdf_writer.write(output_pdf_bytes_io)
            output_pdf_bytes = output_pdf_bytes_io.getvalue()
    
            os.unlink(temp_input_pdf_path) # Clean up temporary input PDF
    
            logger.info(f"OCR completed for {file.filename}. Returning searchable PDF.")
            return StreamingResponse(io.BytesIO(output_pdf_bytes), media_type="application/pdf",
                                     headers={"Content-Disposition": f"attachment; filename=ocr_{file.filename}"})
    
        except Exception as e:
            logger.error(f"Error processing PDF: {e}", exc_info=True)
            raise HTTPException(status_code=500, detail=f"OCR processing failed: {str(e)}")
    ```
    
3.  requirements.txt:
    
    Create a file named requirements.txt in the ~/paddleocr-service directory:
    
    ```
    fastapi
    uvicorn
    paddleocr
    PyMuPDF # for fitz
    Pillow
    reportlab
    PyPDF2
    numpy
    opencv-python-headless # for cv2 if needed by paddleocr, headless for server
    ```
    
4.  Dockerfile:
    
    Create a file named Dockerfile in the ~/paddleocr-service directory:
    
    Dockerfile
    
    ```
    FROM python:3.10-slim-buster
    
    # Install system dependencies for PaddleOCR, PyMuPDF, and ReportLab
    # libgl1-mesa-glx and libglib2.0-0 are common for image processing/rendering
    # fontconfig is needed by reportlab for fonts
    RUN apt-get update && apt-get install -y \
        libgl1-mesa-glx \
        libglib2.0-0 \
        fontconfig \
        # Clean up apt cache to reduce image size
        && rm -rf /var/lib/apt/lists/*
    
    # Set working directory
    WORKDIR /app
    
    # Copy application files
    COPY requirements.txt./
    COPY app.py./
    
    # Install Python dependencies
    RUN pip install --no-cache-dir -r requirements.txt
    
    # Expose the port FastAPI will run on
    EXPOSE 8000
    
    # Command to run the application
    CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
    ```
    
5.  docker-compose.yml:
    
    Create a file named docker-compose.yml in the ~/paddleocr-service directory:
    
    YAML
    
    ```
    version: '3.8'
    
    services:
      paddleocr-service:
        build:.
        container_name: paddleocr-service
        restart: unless-stopped
        ports:
          - "8000:8000" # Expose port 8000 for the OCR service
        volumes:
          # Cache PaddleOCR models to persist downloads
          -./paddleocr_models:/root/.paddleocr/whl/rec
          -./paddleocr_models:/root/.paddleocr/whl/det
          -./paddleocr_models:/root/.paddleocr/whl/cls
        environment:
          PYTHONUNBUFFERED: 1
        # Pass through iGPU devices to this Docker container
        devices:
          - /dev/dri:/dev/dri
    ```
    
6.  Build and Start the PaddleOCR Service:
    
    On your 'docker' VM, navigate to the ~/paddleocr-service directory and run:
    
    Bash
    
    ```
    docker compose up -d --build
    ```
    
    This will build the Docker image, download PaddleOCR models (this might take some time on the first run), and start the service. You can monitor logs with `docker compose logs -f paddleocr-service`.
    

## Step 2: Create the `PAPERLESS_PRE_CONSUME_SCRIPT`

This Python script will run within your Paperless-ngx container. It will send the incoming document to your PaddleOCR service, receive the OCR-enhanced PDF, and overwrite the original document in the consumption directory.

1.  **Install Necessary Python Libraries in Paperless-ngx Container:**
    
    - SSH into your 'docker' VM.
        
    - Access the shell of your Paperless-ngx `webserver` container:
        
        Bash
        
        ```
        docker compose exec webserver bash
        ```
        
    - Install the `requests` library to communicate with the FastAPI service:
        
        Bash
        
        ```
        pip install requests
        ```
        
    - Exit the container shell: `exit`.
        
2.  **Create the `pre-consume-paddleocr.py` Script on the 'docker' VM host:**
    
    - Create a shared script directory on your 'docker' VM host (e.g., `/opt/paperless-scripts`). This directory should be the same one you mounted into your Paperless-ngx `docker-compose.yml` in Guide 1, Step 3.
        
        Bash
        
        ```
        sudo mkdir -p /opt/paperless-scripts
        ```
        
    - Create the `pre-consume-paddleocr.py` script inside this directory:
        
        Bash
        
        ```
        sudo nano /opt/paperless-scripts/pre-consume-paddleocr.py
        ```
        
        Paste the following Python code:
        
        Python
        
        ```
        #!/usr/bin/env python3
        
        import os
        import sys
        import requests
        import logging
        
        # Configure logging to Paperless-ngx logs
        logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
        logger = logging.getLogger(__name__)
        
        def main():
            # DOCUMENT_WORKING_PATH is provided by Paperless-ngx [5]
            document_path = os.environ.get('DOCUMENT_WORKING_PATH')
            if not document_path:
                logger.error("DOCUMENT_WORKING_PATH environment variable not set.")
                sys.exit(1)
        
            # Use the Docker service name for inter-container communication if on the same network.
            # Otherwise, use the IP address of your 'docker' VM.
            paddleocr_service_url = "http://paddleocr-service:8000/ocr-pdf-searchable/"
        
            logger.info(f"Pre-consumption script started for: {document_path}")
        
            if not os.path.exists(document_path):
                logger.error(f"Document not found at: {document_path}")
                sys.exit(1)
        
            if not document_path.lower().endswith('.pdf'):
                logger.info(f"Skipping non-PDF document: {document_path}")
                sys.exit(0)
        
            try:
                with open(document_path, 'rb') as f:
                    # 'file' is the field name expected by the FastAPI service
                    files = {'file': (os.path.basename(document_path), f.read(), 'application/pdf')}
                    logger.info(f"Sending {document_path} to PaddleOCR service at {paddleocr_service_url}")
                    # Set a generous timeout for OCR processing
                    response = requests.post(paddleocr_service_url, files=files, timeout=600) # 10 minute timeout
        
                response.raise_for_status() # Raise an exception for HTTP errors (4xx or 5xx)
        
                if response.headers.get('Content-Type') == 'application/pdf':
                    # Overwrite the original document with the OCR-enhanced PDF [5]
                    with open(document_path, 'wb') as f:
                        f.write(response.content)
                    logger.info(f"Successfully OCR'd and updated: {document_path}")
                else:
                    logger.error(f"PaddleOCR service did not return a PDF. Content-Type: {response.headers.get('Content-Type')}")
                    logger.error(f"Service response (first 500 chars): {response.text[:500]}...")
                    sys.exit(1)
        
            except requests.exceptions.ConnectionError as e:
                logger.error(f"Connection to PaddleOCR service failed: {e}. Is the service running and accessible?")
                sys.exit(1)
            except requests.exceptions.Timeout:
                logger.error(f"PaddleOCR service timed out after 600 seconds for {document_path}. Document might be too complex or service is slow.")
                sys.exit(1)
            except requests.exceptions.RequestException as e:
                logger.error(f"Network or service error during OCR: {e}")
                sys.exit(1)
            except Exception as e:
                logger.error(f"An unexpected error occurred: {e}", exc_info=True)
                sys.exit(1)
        
        if __name__ == "__main__":
            main()
        ```
        
3.  **Make the script executable:**
    
    Bash
    
    ```
    sudo chmod +x /opt/paperless-scripts/pre-consume-paddleocr.py
    ```
    

## Step 3: Final Paperless-ngx Configuration and Restart

Ensure Paperless-ngx is configured to use the script and then restart it.

1.  **Verify `docker-compose.yml` and `.env` for Paperless-ngx:**
    
    - You should have already configured these in Guide 1, Step 3. Double-check that:
        
        - The `scripts` volume is mounted: `-./scripts:/usr/src/paperless/scripts:ro`
            
        - The `PAPERLESS_PRE_CONSUME_SCRIPT` environment variable is set to the correct internal path: `PAPERLESS_PRE_CONSUME_SCRIPT=/usr/src/paperless/scripts/pre-consume-paddleocr.py`
            
        - `PAPERLESS_OCR_MODE=skip` and `PAPERLESS_OCR_SKIP_ARCHIVE_FILE=true` are set in your `.env` to prevent Paperless-ngx from running its own OCR.
            
        - The `devices: - /dev/dri:/dev/dri` section is present under both `webserver` and `consumer` services in `docker-compose.yml`.
            
2.  **Restart your Paperless-ngx stack:**
    
    - From the `/opt/paperless-ngx` directory on your 'docker' VM:
        
        Bash
        
        ```
        docker compose down
        docker compose up -d
        ```
        

Now, when you add a new PDF document to Paperless-ngx's consumption directory, the `pre-consume-paddleocr.py` script will execute. It will send the document to your PaddleOCR service (running on the same VM and leveraging the iGPU), receive the OCR-enhanced searchable PDF, and overwrite the original document before Paperless-ngx proceeds with its standard consumption process. You should observe a significant improvement in your OCR quality.