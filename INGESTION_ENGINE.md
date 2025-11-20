# Document Ingestion Engine - Detailed Design

## Overview

The ingestion engine is responsible for:
1. Accepting multi-format documents (PDF, PPTX, Text, URLs)
2. Parsing and extracting structured content
3. Queueing processing jobs
4. Orchestrating the chunking and embedding pipeline

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    INGESTION CONTROLLER                        │
│  POST /ingest/file                                             │
│  POST /ingest/url                                              │
│  GET  /ingest/status/:jobId                                    │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│                    INGESTION SERVICE                           │
│  - Validate file                                               │
│  - Upload to S3/MinIO                                          │
│  - Create DB record                                            │
│  - Enqueue job                                                 │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│                    BULL QUEUE (Redis)                          │
│  Job: { documentId, fileUrl, fileType }                       │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│                    INGESTION PROCESSOR                         │
│  1. Download file from S3                                      │
│  2. Select parser based on file type                           │
│  3. Parse → Extract text + metadata                            │
│  4. Chunk text                                                 │
│  5. Generate embeddings                                        │
│  6. Store in Qdrant                                            │
│  7. Update status                                              │
└────────────────────────────────────────────────────────────────┘
```

## Parser Implementations

### PDF Parser

**Libraries**:
- Primary: `pdf-parse` (text extraction)
- Fallback: `pdfjs-dist` (Mozilla's PDF.js for complex layouts)
- Tables: `tabula-js` or manual extraction with coordinates
- OCR: `tesseract.js` for scanned PDFs

**Strategy**:
1. Extract text with layout preservation
2. Detect headings via font size/weight analysis
3. Extract tables separately (maintain structure)
4. Identify page boundaries
5. Extract metadata (title, author, creation date)

**Implementation**:
```typescript
// apps/api/src/modules/ingest/parsers/pdf.parser.ts

import * as pdfjsLib from 'pdfjs-dist';
import pdfParse from 'pdf-parse';
import { Injectable, Logger } from '@nestjs/common';

export interface PDFParseResult {
  text: string;
  pages: Array<{
    pageNumber: number;
    text: string;
    headings: string[];
  }>;
  metadata: {
    title?: string;
    author?: string;
    creationDate?: Date;
    pageCount: number;
  };
  tables: Array<{
    pageNumber: number;
    data: string[][];
  }>;
}

@Injectable()
export class PDFParser {
  private readonly logger = new Logger(PDFParser.name);

  async parse(buffer: Buffer): Promise<PDFParseResult> {
    try {
      // Method 1: Fast text extraction
      const basicData = await pdfParse(buffer);

      // Method 2: Detailed extraction with layout
      const detailedData = await this.parseWithPDFJS(buffer);

      return {
        text: basicData.text,
        pages: detailedData.pages,
        metadata: {
          title: basicData.info?.Title,
          author: basicData.info?.Author,
          creationDate: basicData.info?.CreationDate
            ? new Date(basicData.info.CreationDate)
            : undefined,
          pageCount: basicData.numpages,
        },
        tables: detailedData.tables,
      };
    } catch (error) {
      this.logger.error(`PDF parsing failed: ${error.message}`);
      throw error;
    }
  }

  private async parseWithPDFJS(buffer: Buffer): Promise<{
    pages: PDFParseResult['pages'];
    tables: PDFParseResult['tables'];
  }> {
    const loadingTask = pdfjsLib.getDocument({ data: buffer });
    const pdfDocument = await loadingTask.promise;

    const pages: PDFParseResult['pages'] = [];
    const tables: PDFParseResult['tables'] = [];

    for (let i = 1; i <= pdfDocument.numPages; i++) {
      const page = await pdfDocument.getPage(i);
      const textContent = await page.getTextContent();

      // Extract text with font information
      const textItems = textContent.items.map((item: any) => ({
        text: item.str,
        fontSize: item.transform[0],
        fontWeight: item.fontName?.includes('Bold') ? 'bold' : 'normal',
        x: item.transform[4],
        y: item.transform[5],
      }));

      // Detect headings (larger font size)
      const avgFontSize =
        textItems.reduce((sum, item) => sum + item.fontSize, 0) /
        textItems.length;
      const headings = textItems
        .filter((item) => item.fontSize > avgFontSize * 1.3)
        .map((item) => item.text);

      // Combine text
      const pageText = textItems.map((item) => item.text).join(' ');

      pages.push({
        pageNumber: i,
        text: pageText,
        headings,
      });

      // Extract tables (simplified - detect grid patterns)
      const tableData = this.extractTablesFromPage(textItems);
      if (tableData.length > 0) {
        tables.push({
          pageNumber: i,
          data: tableData,
        });
      }
    }

    return { pages, tables };
  }

  private extractTablesFromPage(textItems: any[]): string[][] {
    // Group items by Y coordinate (rows)
    const rows = new Map<number, any[]>();

    textItems.forEach((item) => {
      const rowKey = Math.round(item.y / 5) * 5; // Snap to grid
      if (!rows.has(rowKey)) {
        rows.set(rowKey, []);
      }
      rows.get(rowKey)!.push(item);
    });

    // Sort each row by X coordinate (columns)
    const sortedRows = Array.from(rows.values())
      .map((row) => row.sort((a, b) => a.x - b.x))
      .filter((row) => row.length > 2); // Only keep rows with multiple columns

    // Convert to string matrix
    return sortedRows.map((row) => row.map((item) => item.text));
  }

  async parseWithOCR(buffer: Buffer): Promise<string> {
    // Fallback for scanned PDFs
    // Implementation with tesseract.js
    const { createWorker } = require('tesseract.js');

    const worker = await createWorker('eng');
    // Convert PDF to images first (using pdf2pic or similar)
    // Then run OCR on each image
    // Combine results

    await worker.terminate();
    return '';
  }
}
```

### PPTX Parser

**Libraries**:
- Primary: `pptxgenjs` or `officegen` for parsing
- Alternative: `node-pptx` or unzip + XML parsing

**Strategy**:
1. Extract slide text (title, body, bullet points)
2. Extract speaker notes
3. Preserve slide order
4. Extract embedded text from shapes
5. Capture metadata (title, author, slide count)

**Implementation**:
```typescript
// apps/api/src/modules/ingest/parsers/pptx.parser.ts

import JSZip from 'jszip';
import * as xml2js from 'xml2js';
import { Injectable, Logger } from '@nestjs/common';

export interface PPTXParseResult {
  slides: Array<{
    slideNumber: number;
    title: string;
    content: string[];
    notes: string;
    shapes: Array<{
      type: string;
      text: string;
    }>;
  }>;
  metadata: {
    title?: string;
    author?: string;
    slideCount: number;
  };
}

@Injectable()
export class PPTXParser {
  private readonly logger = new Logger(PPTXParser.name);

  async parse(buffer: Buffer): Promise<PPTXParseResult> {
    try {
      const zip = await JSZip.loadAsync(buffer);

      // Extract metadata from core.xml
      const metadata = await this.extractMetadata(zip);

      // Extract slides
      const slides = await this.extractSlides(zip);

      return {
        slides,
        metadata: {
          ...metadata,
          slideCount: slides.length,
        },
      };
    } catch (error) {
      this.logger.error(`PPTX parsing failed: ${error.message}`);
      throw error;
    }
  }

  private async extractMetadata(zip: JSZip) {
    const coreXml = await zip.file('docProps/core.xml')?.async('string');
    if (!coreXml) return {};

    const parser = new xml2js.Parser();
    const result = await parser.parseStringPromise(coreXml);

    return {
      title: result['cp:coreProperties']?.['dc:title']?.[0] || '',
      author: result['cp:coreProperties']?.['dc:creator']?.[0] || '',
    };
  }

  private async extractSlides(zip: JSZip): Promise<PPTXParseResult['slides']> {
    const slides: PPTXParseResult['slides'] = [];

    // Get slide files (slide1.xml, slide2.xml, etc.)
    const slideFiles = Object.keys(zip.files)
      .filter((filename) => /ppt\/slides\/slide\d+\.xml/.test(filename))
      .sort((a, b) => {
        const numA = parseInt(a.match(/\d+/)?.[0] || '0');
        const numB = parseInt(b.match(/\d+/)?.[0] || '0');
        return numA - numB;
      });

    for (let i = 0; i < slideFiles.length; i++) {
      const slideXml = await zip.file(slideFiles[i])?.async('string');
      if (!slideXml) continue;

      const parser = new xml2js.Parser();
      const slideData = await parser.parseStringPromise(slideXml);

      // Extract text from shapes
      const shapes = this.extractShapes(slideData);

      // Find title (usually first shape)
      const title = shapes.find((s) => s.type === 'title')?.text || '';

      // Get speaker notes
      const notesFile = slideFiles[i].replace('/slides/', '/notesSlides/').replace('slide', 'notesSlide');
      const notes = await this.extractNotes(zip, notesFile);

      slides.push({
        slideNumber: i + 1,
        title,
        content: shapes.filter((s) => s.type !== 'title').map((s) => s.text),
        notes,
        shapes,
      });
    }

    return slides;
  }

  private extractShapes(slideData: any): Array<{ type: string; text: string }> {
    const shapes: Array<{ type: string; text: string }> = [];

    // Navigate XML structure: sld:sld -> sld:cSld -> sld:spTree -> sld:sp (shapes)
    const spTree =
      slideData?.['p:sld']?.[0]?['p:cSld']?.[0]?['p:spTree']?.[0];
    if (!spTree) return shapes;

    const shapeElements = spTree['p:sp'] || [];

    shapeElements.forEach((shape: any, index: number) => {
      // Extract text from shape
      const textBody = shape['p:txBody']?.[0];
      if (!textBody) return;

      const paragraphs = textBody['a:p'] || [];
      const textParts: string[] = [];

      paragraphs.forEach((para: any) => {
        const runs = para['a:r'] || [];
        runs.forEach((run: any) => {
          const text = run['a:t']?.[0];
          if (text) textParts.push(text);
        });
      });

      const combinedText = textParts.join(' ').trim();
      if (combinedText) {
        // Detect if it's a title based on position or placeholder type
        const placeholderType =
          shape['p:nvSpPr']?.[0]?.['p:nvPr']?.[0]?.['p:ph']?.[0]?.$?.type;

        shapes.push({
          type: placeholderType === 'title' || index === 0 ? 'title' : 'content',
          text: combinedText,
        });
      }
    });

    return shapes;
  }

  private async extractNotes(zip: JSZip, notesFile: string): Promise<string> {
    const notesXml = await zip.file(notesFile)?.async('string');
    if (!notesXml) return '';

    const parser = new xml2js.Parser();
    const notesData = await parser.parseStringPromise(notesXml);

    // Extract text from notes
    const textParts: string[] = [];
    const spTree =
      notesData?.['p:notes']?.[0]?.['p:cSld']?.[0]?.['p:spTree']?.[0];

    if (spTree?.['p:sp']) {
      spTree['p:sp'].forEach((shape: any) => {
        const textBody = shape['p:txBody']?.[0];
        if (textBody?.['a:p']) {
          textBody['a:p'].forEach((para: any) => {
            const runs = para['a:r'] || [];
            runs.forEach((run: any) => {
              const text = run['a:t']?.[0];
              if (text) textParts.push(text);
            });
          });
        }
      });
    }

    return textParts.join(' ').trim();
  }
}
```

### Text Parser

**Strategy**:
- Simple UTF-8 text extraction
- Detect encoding (UTF-8, ISO-8859-1, etc.)
- Basic cleaning (remove control characters)
- Markdown preservation if detected

**Implementation**:
```typescript
// apps/api/src/modules/ingest/parsers/text.parser.ts

import { Injectable } from '@nestjs/common';
import * as iconv from 'iconv-lite';
import * as jschardet from 'jschardet';

export interface TextParseResult {
  text: string;
  encoding: string;
  lineCount: number;
  isMarkdown: boolean;
}

@Injectable()
export class TextParser {
  parse(buffer: Buffer): TextParseResult {
    // Detect encoding
    const detected = jschardet.detect(buffer);
    const encoding = detected.encoding || 'UTF-8';

    // Decode text
    let text = iconv.decode(buffer, encoding);

    // Clean text
    text = this.cleanText(text);

    // Detect if markdown
    const isMarkdown = this.isMarkdownContent(text);

    return {
      text,
      encoding,
      lineCount: text.split('\n').length,
      isMarkdown,
    };
  }

  private cleanText(text: string): string {
    // Remove null bytes and control characters (except newlines, tabs)
    return text.replace(/[\x00-\x08\x0B\x0C\x0E-\x1F]/g, '');
  }

  private isMarkdownContent(text: string): boolean {
    // Simple heuristic: check for markdown patterns
    const markdownPatterns = [
      /^#{1,6}\s/m, // Headings
      /\*\*.*\*\*/,  // Bold
      /_.*_/,        // Italic
      /\[.*\]\(.*\)/, // Links
      /^[-*+]\s/m,   // Lists
      /^>\s/m,       // Blockquotes
      /```/,         // Code blocks
    ];

    return markdownPatterns.some((pattern) => pattern.test(text));
  }
}
```

### URL Parser (Web Scraper)

**Libraries**:
- `axios` for HTTP requests
- `cheerio` for HTML parsing
- `@mozilla/readability` for article extraction
- `turndown` for HTML to Markdown conversion

**Strategy**:
1. Fetch URL with proper headers
2. Extract main content (article, not navigation/ads)
3. Convert HTML to clean markdown
4. Extract metadata (title, author, date, description)
5. Handle JavaScript-rendered pages (optional: Puppeteer)

**Implementation**:
```typescript
// apps/api/src/modules/ingest/parsers/url.parser.ts

import axios from 'axios';
import * as cheerio from 'cheerio';
import { Readability } from '@mozilla/readability';
import { JSDOM } from 'jsdom';
import TurndownService from 'turndown';
import { Injectable, Logger } from '@nestjs/common';

export interface URLParseResult {
  content: string;
  markdown: string;
  metadata: {
    title: string;
    author?: string;
    publishedDate?: Date;
    description?: string;
    url: string;
    siteName?: string;
  };
}

@Injectable()
export class URLParser {
  private readonly logger = new Logger(URLParser.name);
  private readonly turndownService = new TurndownService({
    headingStyle: 'atx',
    codeBlockStyle: 'fenced',
  });

  async parse(url: string): Promise<URLParseResult> {
    try {
      // Fetch URL with browser-like headers
      const response = await axios.get(url, {
        headers: {
          'User-Agent':
            'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
          Accept: 'text/html,application/xhtml+xml,application/xml;q=0.9',
        },
        timeout: 30000,
        maxRedirects: 5,
      });

      const html = response.data;

      // Parse with Readability (extracts main content)
      const dom = new JSDOM(html, { url });
      const reader = new Readability(dom.window.document);
      const article = reader.parse();

      if (!article) {
        throw new Error('Failed to extract article content');
      }

      // Convert HTML to Markdown
      const markdown = this.turndownService.turndown(article.content);

      // Extract metadata with Cheerio
      const $ = cheerio.load(html);
      const metadata = this.extractMetadata($, url);

      return {
        content: article.textContent,
        markdown,
        metadata: {
          ...metadata,
          title: article.title || metadata.title,
        },
      };
    } catch (error) {
      this.logger.error(`URL parsing failed for ${url}: ${error.message}`);
      throw error;
    }
  }

  private extractMetadata(
    $: cheerio.CheerioAPI,
    url: string,
  ): URLParseResult['metadata'] {
    // Open Graph tags
    const ogTitle = $('meta[property="og:title"]').attr('content');
    const ogDescription = $('meta[property="og:description"]').attr('content');
    const ogSiteName = $('meta[property="og:site_name"]').attr('content');
    const ogPublishedTime = $('meta[property="article:published_time"]').attr(
      'content',
    );
    const ogAuthor = $('meta[property="article:author"]').attr('content');

    // Standard meta tags
    const metaDescription = $('meta[name="description"]').attr('content');
    const metaAuthor = $('meta[name="author"]').attr('content');

    // Fallbacks
    const title =
      ogTitle || $('title').text() || $('h1').first().text() || 'Untitled';

    return {
      title,
      author: ogAuthor || metaAuthor,
      publishedDate: ogPublishedTime ? new Date(ogPublishedTime) : undefined,
      description: ogDescription || metaDescription,
      url,
      siteName: ogSiteName,
    };
  }
}
```

## OCR Engine (Optional)

**Libraries**:
- `tesseract.js` for in-browser/Node OCR
- Alternative: Google Cloud Vision API, AWS Textract

**Implementation**:
```typescript
// apps/api/src/modules/ingest/parsers/ocr.engine.ts

import { createWorker } from 'tesseract.js';
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class OCREngine {
  private readonly logger = new Logger(OCREngine.name);

  async extractTextFromImage(imageBuffer: Buffer): Promise<string> {
    const worker = await createWorker('eng', 1, {
      logger: (m) => this.logger.debug(m),
    });

    try {
      const {
        data: { text },
      } = await worker.recognize(imageBuffer);
      return text;
    } finally {
      await worker.terminate();
    }
  }

  async extractTextFromPDFPages(
    pdfBuffer: Buffer,
    pages: number[],
  ): Promise<Map<number, string>> {
    // Convert PDF pages to images (using pdf2pic or similar)
    // Then run OCR on each image
    const results = new Map<number, string>();

    // Implementation here...

    return results;
  }
}
```

## Integration: Ingestion Service

```typescript
// apps/api/src/modules/ingest/ingest.service.ts

import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';
import { Document } from '../../database/entities/document.entity';
import { S3Service } from '../../shared/services/s3.service';
import { PDFParser } from './parsers/pdf.parser';
import { PPTXParser } from './parsers/pptx.parser';
import { TextParser } from './parsers/text.parser';
import { URLParser } from './parsers/url.parser';

@Injectable()
export class IngestService {
  private readonly logger = new Logger(IngestService.name);

  constructor(
    @InjectRepository(Document)
    private readonly documentRepo: Repository<Document>,
    @InjectQueue('ingestion')
    private readonly ingestionQueue: Queue,
    private readonly s3Service: S3Service,
    private readonly pdfParser: PDFParser,
    private readonly pptxParser: PPTXParser,
    private readonly textParser: TextParser,
    private readonly urlParser: URLParser,
  ) {}

  async ingestFile(
    userId: string,
    file: Express.Multer.File,
  ): Promise<Document> {
    // 1. Validate file
    this.validateFile(file);

    // 2. Calculate content hash
    const contentHash = this.calculateHash(file.buffer);

    // 3. Check for duplicates
    const existing = await this.documentRepo.findOne({
      where: { userId, contentHash },
    });

    if (existing) {
      this.logger.log(`Duplicate file detected: ${contentHash}`);
      return existing;
    }

    // 4. Upload to S3
    const fileUrl = await this.s3Service.upload(
      `${userId}/${Date.now()}-${file.originalname}`,
      file.buffer,
      file.mimetype,
    );

    // 5. Create document record
    const document = this.documentRepo.create({
      userId,
      title: file.originalname,
      fileName: file.originalname,
      fileType: this.getFileType(file.mimetype),
      fileSize: file.size,
      fileUrl,
      contentHash,
      status: 'pending',
    });

    await this.documentRepo.save(document);

    // 6. Enqueue processing job
    await this.ingestionQueue.add('process-document', {
      documentId: document.id,
      fileUrl,
      fileType: document.fileType,
    });

    this.logger.log(`Document ${document.id} enqueued for processing`);

    return document;
  }

  async ingestURL(userId: string, url: string): Promise<Document> {
    // Similar to ingestFile but fetches URL first
    // Then creates document and enqueues job
    // Implementation here...
  }

  private validateFile(file: Express.Multer.File): void {
    const maxSize = 100 * 1024 * 1024; // 100MB
    if (file.size > maxSize) {
      throw new Error('File too large');
    }

    const allowedMimeTypes = [
      'application/pdf',
      'application/vnd.openxmlformats-officedocument.presentationml.presentation',
      'text/plain',
      'text/markdown',
    ];

    if (!allowedMimeTypes.includes(file.mimetype)) {
      throw new Error('Unsupported file type');
    }
  }

  private calculateHash(buffer: Buffer): string {
    const crypto = require('crypto');
    return crypto.createHash('sha256').update(buffer).digest('hex');
  }

  private getFileType(mimetype: string): string {
    const mimeTypeMap: Record<string, string> = {
      'application/pdf': 'pdf',
      'application/vnd.openxmlformats-officedocument.presentationml.presentation':
        'pptx',
      'text/plain': 'txt',
      'text/markdown': 'md',
    };

    return mimeTypeMap[mimetype] || 'unknown';
  }
}
```

## Next: Chunking System

The parsed content will be passed to the chunking service, which will apply various strategies to break it into optimal chunks for embedding and retrieval.
