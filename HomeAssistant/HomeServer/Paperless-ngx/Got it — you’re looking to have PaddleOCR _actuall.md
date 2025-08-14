---
title: >-
  Got it — you’re looking to have PaddleOCR *actually perform the text
  recognition
updated: 2025-08-10 20:13:17Z
created: 2025-08-10 20:13:15Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

Got it — you’re looking to have PaddleOCR *actually perform the text recognition*, embed that recognized text layer into the PDF, and then pass that “searchable” PDF on to Paperless‑ngx, where Tesseract can still run (or skip if you disable it).

This changes the workflow: instead of PaddleOCR just detecting and deskewing, it now *replaces* the initial OCR step. Think of it like pre‑baking the text layer into the PDF before Paperless sees it.

Here’s a **markdown guide** to set this up in your Docker‑based Debian VM, building on your current Paperless‑ngx deployment.

* * *

# 📄 Using PaddleOCR to OCR & Embed Text Before Paperless‑ngx/Tesseract

## **Overview**

The plan:

1.  Convert incoming files to images (per page).
    
2.  Run PaddleOCR on each page (full text recognition).
    
3.  Generate a searchable PDF from images + PaddleOCR text (using `reportlab` or `ocrmypdf`\-style workflow).
    
4.  Hand the processed PDF to Paperless as if it were the original input.
    

* * *

## **Step 1 — Prepare a custom Paperless image**

We’ll extend the Paperless image to add PaddleOCR and a PDF text-layer builder.

**Dockerfile:**

```
FROM ghcr.io/paperless-ngx/paperless-ngx:latest

USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils \
    python3-reportlab \
    && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir \
    paddlepaddle==2.6.1 \
    paddleocr==2.7.0.3

COPY paddle_textocr.py /opt/paddle_textocr.py
COPY preconsume_paddleocr.sh /usr/local/bin/preconsume_paddleocr.sh
RUN chmod +x /usr/local/bin/preconsume_paddleocr.sh /opt/paddle_textocr.py

USER 1000
```

* * *

## **Step 2 — Shell wrapper for Paperless**

`preconsume_paddleocr.sh`:

```
#!/usr/bin/env bash
set -euo pipefail

IN="$1"
WORKDIR="$(mktemp -d /tmp/paddleocrproc.XXXXXX)"

if OUT="$(python3 /opt/paddle_textocr.py "$IN" "$WORKDIR")"; then
  if [ -n "$OUT" ] && [ -f "$OUT" ]; then
    echo "$OUT"
  fi
fi
exit 0
```

* * *

## **Step 3 — PaddleOCR + Text Layer Embedder**

`paddle_textocr.py`:

```
#!/usr/bin/env python3
import sys, os, glob, subprocess
from pathlib import Path
from paddleocr import PaddleOCR
import fitz  # PyMuPDF
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.lib.units import inch
import tempfile

def pdf_to_images(pdf_path, out_dir, dpi=300):
    subprocess.run(["pdftoppm", "-r", str(dpi), "-png", pdf_path, os.path.join(out_dir, "page")], check=True)
    return sorted(glob.glob(os.path.join(out_dir, "page-*.png")))

def create_searchable_pdf(images, ocr_results, out_path):
    # Use PyMuPDF to draw images, then overlay text in invisible layer
    pdf = fitz.open()
    for img_path, results in zip(images, ocr_results):
        img_pix = fitz.Pixmap(img_path)
        page = pdf.new_page(width=img_pix.width, height=img_pix.height)
        page.insert_image(page.rect, filename=img_path)
        for line in results:
            box, (txt, conf) = line
            if not txt.strip():
                continue
            # Convert box coords to rectangle
            x0, y0 = box[0]
            x2, y2 = box[2]
            rect = fitz.Rect(x0, y0, x2, y2)
            page.insert_textbox(rect, txt, fontsize=10, color=(1,1,1,0), render_mode=3)  # Invisible
    pdf.save(out_path)

def process(in_path, workdir):
    images = pdf_to_images(in_path, workdir, dpi=300)
    ocr = PaddleOCR(use_angle_cls=True, lang='en')  # adjust lang
    results_per_page = []
    for img in images:
        res = ocr.ocr(img, cls=True)
        # res[0] is list of (box, (text, conf))
        results_per_page.append(res[0] if res else [])
    out_pdf = os.path.join(workdir, "paddleocr_out.pdf")
    create_searchable_pdf(images, results_per_page, out_pdf)
    return out_pdf

def main():
    if len(sys.argv) < 3:
        sys.exit(0)
    in_path, workdir = sys.argv[1], sys.argv[2]
    Path(workdir).mkdir(parents=True, exist_ok=True)
    try:
        out_pdf = process(in_path, workdir)
        print(out_pdf, end="")
    except Exception as e:
        pass

if __name__ == "__main__":
    main()
```

* * *

## **Step 4 — Hook into docker-compose**

[`docker-compose.override.yml`](https://docker-compose.override.yml):

```
services:
  paperless:
    build:
      context: ./paperless-paddle
    environment:
      - PAPERLESS_PRE_CONSUME_SCRIPT=/usr/local/bin/preconsume_paddleocr.sh
```

* * *

## **Step 5 — Build & run**

```
cd /opt/paperless
docker compose build paperless
docker compose up -d
```

* * *

## **Step 6 — Test**

Drop a scanned PDF into your consume folder, then inspect in Paperless:

- You should see a text layer already there from PaddleOCR.
    
- Tesseract will still run unless disabled; you can A/B test by disabling it via `PAPERLESS_OCR_SKIP_ARCHIVE_FILE` or related flags.
    

* * *

### 🔧 Notes

- PaddleOCR language packs: change `lang='en'` to `ch`, `fr`, `de`, etc., or multi‑stage if needed.
    
- PyMuPDF (`fitz`) is used for precise placement of invisible text over the scanned image.
    
- If you want PaddleOCR’s output *instead of* Tesseract’s, you can disable Tesseract in Paperless via environment variables.
    

* * *

If you’d like, I can adapt this to **completely bypass Tesseract** and only store PaddleOCR text — that would speed up processing further, and avoid double OCR. Would you like me to give you that stripped‑down config?