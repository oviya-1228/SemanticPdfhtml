# Semantics-Preserving PDF-to-HTML Conversion

### Fusing YOLOv12 Layout Detection with PDF Text Geometry

A hybrid PDF-to-HTML conversion system that aims to preserve both the **visual appearance** and **semantic structure** of PDF documents. It combines deep-learning-based document layout detection with precise PDF text geometry to generate structured, visually faithful, and machine-readable HTML.

## Overview

PDFs preserve the appearance of documents but often lack the semantic structure required for accessibility, content extraction, search, and downstream processing.

This project addresses that limitation through a dual-path conversion pipeline:

- **Visual path:** YOLOv12-L, fine-tuned on DocLayNet, identifies document layout regions.
- **Geometry path:** PyMuPDF extracts text spans, bounding boxes, font information, sizes, styles, and colours.

The outputs of these paths are combined to identify the semantic role of each text block while retaining its original position and appearance.

The system generates positioned HTML, semantic HTML, and structured JSON representations. It also supports table extraction, spreadsheet export, and human-in-the-loop correction.

## Key Features

- **Document Layout Detection:** Uses a YOLOv12-L detector trained on DocLayNet to identify document regions.
- **Geometry-Based Text Placement:** Uses PDF text coordinates and span-level properties to preserve positioning and styling.
- **Semantic HTML Generation:** Converts detected regions into meaningful HTML elements such as headings, paragraphs, lists, tables, and links.
- **Hybrid Block-to-Region Matching:** Uses Intersection over Union (IoU) with a smaller-region tie-break strategy to associate text blocks with detected layout regions.
- **Table Extraction:** Detects tables, reconstructs their cell structure, and extracts content into structured records.
- **Excel Export:** Supports querying extracted table data and exporting selected rows and columns to Excel.
- **Human-in-the-Loop Review:** Provides a side-by-side PDF and HTML comparison interface for correcting extracted text and semantic tags.
- **Multiple Output Formats:** Produces positioned HTML, semantic HTML, and structured JSON.

## System Architecture

The system processes each uploaded PDF through a series of stages.

1. **PDF Validation:** Checks file size, page count, and availability of an extractable text layer.
2. **Parallel Processing:**
   - The PDF page is rendered as an image at 200 DPI for layout detection.
   - PyMuPDF extracts text blocks, lines, spans, bounding boxes, fonts, and colours.
3. **Layout Detection:** YOLOv12-L identifies layout regions using DocLayNet classes.
4. **Block–Region Fusion:** Associates extracted text blocks with detected regions using IoU and a smaller-region tie-break.
5. **Semantic Labelling:** Assigns HTML tags using a three-tier cascade.
6. **Rendering and Extraction:** Generates the HTML outputs and structured JSON, while recovering tables.
7. **Review and Correction:** Allows users to inspect and modify the generated output through the web interface.

## Methodology

### 1. Document Layout Detection

Each PDF page is rendered into an RGB image at 200 DPI and passed to a YOLOv12-L detector.

The detector identifies eleven DocLayNet layout classes, including:

- Title
- Section-header
- List-item
- Table
- Picture
- Caption
- Formula
- Footnote
- Page-header
- Page-footer
- Text

The detection confidence threshold is set to 0.3.

### 2. PDF Text Geometry Extraction

PyMuPDF extracts text blocks, lines, and spans along with their:

- Bounding-box coordinates
- Font names and sizes
- Bold and italic properties
- Text colours

It also supports the extraction of embedded images, icons, filled vector rectangles, and font information.

Unlike the detector, which is responsible for classifying layout regions, PDF geometry is used for the actual placement and styling of text.

### 3. Block-to-Region Fusion

Extracted text blocks are mapped into the detector's pixel coordinate space and matched with predicted layout regions.

The matching process uses Intersection over Union:

\[
IoU(B,D_i)=\frac{|B\cap D_i|}{|B\cup D_i|}
\]

where \(B\) represents a text block and \(D_i\) represents a detected layout region.

The region with the highest IoU is selected. When multiple regions have the same score, the smaller region is preferred because it generally provides more specific semantic information.

If no detection overlaps a text block, it retains the default `Text` classification.

### 4. Semantic Labelling Cascade

Each text block is assigned an HTML tag through a three-tier cascade:

1. **Detector class:** Layout classes are mapped to corresponding HTML elements. For example, Title maps to `h1`, Section-header to `h2`, and List-item to `li`.
2. **List-marker detection:** Blocks classified as plain text are checked for bullet symbols and ordered-list prefixes. Matching blocks are converted into list items.
3. **Relative font-size ranking:** The three largest distinct font sizes in the document are used to identify heading levels. Blocks containing at most twelve words are assigned `h1`, `h2`, or `h3` based on their font-size rank. Remaining blocks become paragraphs.

Consecutive list items are grouped into a single list.

### 5. Visual Fidelity and HTML Rendering

The positioned HTML output uses the original PDF text geometry for element placement.

The rendering process preserves:

- Embedded fonts through CSS `@font-face` rules
- Span-level font size, weight, style, and colour
- Text block positioning
- Line-height and text alignment
- Background fills and embedded images
- Small decorative images and icons

The system also generates semantic HTML without absolute positioning, providing a clean document flow with meaningful structural tags.

### 6. Table Recovery and Structured Extraction

Tables are detected using PyMuPDF's table finder, with pdfplumber's line-based strategy as a fallback.

The extraction process:

- Reconstructs table grids using cell geometry.
- Extracts rich text from individual cells.
- Preserves applicable cell background fills.
- Normalises tables into structured records.
- Builds header aliases for column matching.

Users can query extracted tables using constrained natural-language requests, filter rows, select columns, and export the resulting data into Excel files.

### 7. Human-in-the-Loop Review

The web interface provides a side-by-side comparison of the original PDF and generated HTML.

Reviewers can:
- Correct extracted text.
- Modify semantic HTML tags.
- Inspect page-level word, image, and table statistics.
- Save corrections and regenerate the corresponding JSON representation.

## Technology Stack

| Component | Technology |
|---|---|
| Programming Language | Python |
| Web Framework | Flask |
| Layout Detection | YOLOv12-L |
| Layout Dataset | DocLayNet |
| PDF Processing | PyMuPDF |
| Table Extraction | pdfplumber |
| HTML Processing | BeautifulSoup |
| Font Processing | fontTools |
| Object Detection Runtime | Ultralytics |
| Output Formats | HTML, JSON, Excel |

## Output Formats

The system produces three primary document representations:

| Output | Description |
|---|---|
| Positioned HTML | Preserves PDF coordinates, visual styling, and semantic tags. |
| Semantic HTML | Generates a clean document flow with meaningful headings, paragraphs, lists, and tables. |
| Structured JSON | Stores document pages and extracted elements in a machine-readable representation. |

Additionally, extracted table data can be queried and exported to Excel.

## Current Limitations

The current implementation has the following limitations:

- **Page-boundary structures:** Lists, tables, and sections that continue across pages are not automatically merged.
- **Heading consistency:** Heading levels are ranked per page, which can produce inconsistent heading tags across a document.
- **Nested lists:** List nesting depth and ordered/unordered list distinctions are not reliably recovered.
- **Split inline elements:** Hyperlinks and phrases spanning multiple lines may be split.
- **Scanned PDFs:** Documents without an extractable text layer are rejected because OCR is not currently implemented.
- **Formulas and figures:** Formula regions are emitted as paragraphs, and captions are not automatically associated with figures.

## Future Improvements

Planned directions for improving the system include:

- Cross-page continuation detection for lists, tables, and sections.
- Document-wide heading-size ranking.
- Improved nested-list reconstruction using indentation and marker patterns.
- Better hyperlink reconstruction using PDF link annotations.
- OCR integration for scanned documents.
- MathML generation for mathematical expressions.
- Automatic caption-to-figure association.
- Evaluation against established document-conversion systems.

## Reproducibility

## Reproducibility

The results can be reproduced using the following steps:

1. Clone the repository:
   ```bash
   git clone https://github.com/oviya-1228/SemanticPdfhtml
   cd pdfhtml
   ```

2. Install the required dependencies:
   ```bash
   npm install
   ```

3. Run the application:
   ```bash
   npm start
   ```

4. Open the application in a browser and perform the same input steps described in the project evaluation or demo instructions.

For consistent results:

- Use the dependency versions recorded in `package-lock.json`.
- Use the same operating system, browser, PDF or HTML input, and browser viewport settings.
- Restart the application between test runs.
- Check that the generated output differs only according to the tested configuration or input.

If `npm install` fails, use the repository's pinned package lock and Node.js version where available.


## Project Team

**Department of Information Technology**  
Sri Sivasubramaniya Nadar College of Engineering, Chennai, India.

- Sarveash Koushik R
- Oviya T S
- Adithiya T
- Mohnish Kanna K S

**Faculty Contributors**
- Dr. K. R. Uthayan
- Dr. N. Sripriya

---

*Semantics-preserving document conversion through the integration of visual layout understanding and PDF text geometry.*
