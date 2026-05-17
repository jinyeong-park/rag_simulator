# System Prompt: Build a RAG Process Visualizer (Single HTML File)

## Goal
Build a fully functional, interactive **RAG (Retrieval-Augmented Generation) Process Visualizer** as a **single self-contained HTML file** (HTML + CSS + JS, no external frameworks, no backend, no API calls). Everything runs in the browser.

---

## Design System

### Fonts
- Body/UI: `DM Sans` (Google Fonts, weights 400/500/600)
- Monospace (chunk text, vectors, code): `DM Mono` (Google Fonts, weights 400/500)

### Color Variables (CSS :root)
```css
--blue: #3B6FF0;
--blue-light: #EEF2FD;
--purple: #8B5CF6;
--purple-light: #F3EFFE;
--green: #22C55E;
--green-light: #EDFAF3;
--orange: #F97316;
--orange-light: #FFF3EA;
--pink: #EC4899;
--pink-light: #FEF0F7;
--gray-50: #F8F9FB;
--gray-100: #F0F2F7;
--gray-200: #E2E6EF;
--gray-300: #C7CDD8;
--gray-400: #94A0B4;
--gray-600: #5A6478;
--gray-800: #1E2330;
```

### Layout
- Page background: `--gray-50`
- Sticky top progress bar (height 56px, white bg, bottom border)
- Hero section: centered title in `--blue`, subtitle in `--gray-600`
- Main content: CSS Grid, `grid-template-columns: 420px 1fr`, gap 24px, max-width 1200px, centered
- Cards: white bg, `border-radius: 16px`, `border: 1px solid var(--gray-200)`, subtle box-shadow

---

## Layout Structure

### Top: Progress Bar (sticky)
5 steps in a row with connecting lines:
`1 Chunk → 2 Embed → 3 Save → 4 Query → 5 Answer`

- Each step has a circle (28px) + label
- Circle states: default (gray), active (blue), done (green with ✓)
- Connecting lines turn green when the step before them is done
- Update dynamically as user progresses

### Hero (below progress bar)
- Title: "RAG Process Visualizer" in `--blue`, 30px, font-weight 600
- Subtitle: "Interactive demonstration of the Retrieval-Augmented Generation pipeline. Process your documents through chunking, embedding, and intelligent querying."

---

## Left Column (420px): Control Panels

### Card 1 — Document Input & Chunking (blue badge "1")
- Textarea: "Paste your document content here (articles, handbooks, FAQs, transcripts, etc.)"
- Character count shown below textarea (right-aligned, updates live)
- Minimum 40 characters validation
- Two number inputs in a 2-column grid:
  - **Chunk Size** (default: 350) — "Characters per chunk"
  - **Overlap** (default: 80) — "Characters overlap between chunks"
- Three buttons (full width, stacked, 10px margin-top each):
  1. **Chunk Document** — blue (`#3B6FF0`)
  2. **Create Embeddings** — purple (`#8B5CF6`), disabled until chunking done
  3. **Save to Vector DB** — green (`#16A34A`), disabled until embeddings done

### Card 2 — Query & Retrieval (orange badge "2")
- Textarea: "What are the key points mentioned in the document?"
- Two selects in a 2-column grid:
  - **Retrieve Top N**: options 1–5 chunks (default: 3)
  - **Answer Mode**: "Offline (Extractive)" / "Online (Generative)"
- Three buttons (stacked):
  1. **Retrieve Relevant Chunks** — orange, disabled until Vector DB saved
  2. **Compose Query** — dark gray (`#5A6478`), disabled until retrieval done
  3. **Generate Answer** — pink (`#EC4899`), disabled until query composed

---

## Right Column: Output Panels (flex column, gap 24px)

### Panel 1 — Process Status
Simple card with 5 rows:
- Document Chunked
- Embeddings Created
- Vector DB Ready
- Query Executed
- Answer Generated

Each row has a circle icon on the right. Default: gray border. Done state: green background + green border + checkmark SVG.

### Panel 2 — Document Chunks
- Header: green dot + "Document Chunks" + chunk count (e.g. "5 chunks") on right
- After chunking: show overlap legend at top (3 colored squares):
  - Blue = overlap from previous chunk
  - Yellow = overlap into next chunk  
  - Purple = both overlaps
- Each chunk card shows:
  - Header row: `#001` number badge + overlap tags (`↩ prev overlap: 80c`, `next overlap: 80c ↪`) + character count on far right
  - Full chunk text below with colored highlights:
    - First N chars (prev overlap): `background: #DBEAFE` (blue)
    - Last N chars (next overlap): `background: #FEF3C7` (yellow)
    - Both: `background: #EDE9FE` (purple)
  - Click to highlight (orange border + orange background)
- Scrollable, max-height 280px

### Panel 3 — Embeddings & Metadata
- Header: purple dot + "Embeddings & Metadata" + "Vector representations of your document chunks"
- After creating embeddings: render as a **table** with columns:
  - **Chunk** — `#0`, `#1`... with a clickable `jump` link in purple that scrolls to that chunk
  - **Top terms** — top 6 most frequent non-stopword terms from that chunk, comma-separated
  - **Dims** — integer, calculated from chunk word count × 4 + 10 (capped at 1536)
  - **Spark** — inline SVG polyline (120×24px) showing the simulated embedding vector as a sparkline, stroke color `#6D28D9`, stroke-width 1.2

### Panel 4 — Retrieval Results
- Header: orange dot + "Retrieval Results" + "Most relevant chunks for your query"
- After retrieval: show ranked result cards (orange-tinted background `#FFF3EA`, border `#FED7AA`):
  - Header row: `Chunk #N` + clickable `view` link (blue, underlined) | `cos 0.XXX` pill (gray bg, monospace font, rounded 6px border)
  - Body: first 220 chars of chunk text in monospace, truncated with `…`

### Panel 5 & 6 — Bottom Row (2-column grid)

**Composed Query** (left):
- Header: edit icon SVG + title "Composed Query" + subtitle "Final prompt sent to the LLM with context"
- After composing: show a scrollable monospace text box with the full prompt in this exact format:
```
Answer the following user query based only on the provided data.
user_query: <the user's question>

Data:
[[chunk_N]]
<full chunk text>

[[chunk_M]]
<full chunk text>
```

**AI Answer** (right):
- Header: pink speech-bubble SVG icon (left) + layout with:
  - Left: big bold "AI\nAnswer" (18px, weight 700, two lines)
  - Right: `Top N: 3 • Retrieved from local vector DB` in gray-400, 12px (dynamically updated with selected Top N value)
  - Below both: "Generated response based on your document" in gray-400
- After generating: show answer in a pink-tinted box (`#FEF0F7`, border `#FBCFE8`)

---

## JavaScript Logic

### Chunking
```js
// Split text into overlapping chunks
let i = 0;
const starts = [];
while (i < text.length) {
  starts.push(i);
  i += chunkSize - overlap;
  if (i + overlap >= text.length && i < text.length) {
    starts.push(i);
    break;
  }
}
// For each chunk, track prevOverlapLen and nextOverlapLen for highlighting
```

### Simulated Embeddings
```js
// Simple deterministic pseudo-random vector
function simulateEmbeddingVector(seed) {
  const vec = [];
  let x = seed * 9301 + 49297;
  for (let i = 0; i < 40; i++) {
    x = (x * 9301 + 49297) % 233280;
    vec.push(x / 233280);
  }
  return vec;
}
```

### Top Terms Extraction
- Lowercase the chunk text, remove non-alpha chars, split on whitespace
- Filter out stopwords (the, and, for, are, but, not, you, all, can, that, this, with, have, from, they, will, been, their, there, were, when, what, etc.)
- Count frequency, return top 6 words

### Scoring / Retrieval
Combine lexical + cosine similarity:
```js
function scoreChunkForQuery(chunk, query) {
  // lexical: what % of query words (>3 chars) appear in chunk
  // cosine: cosineSimilarity(textToVector(chunk), textToVector(query))
  // where textToVector hashes the text to a seed and calls simulateEmbeddingVector
  return Math.min(1, 0.45 * lexical + 0.55 * cosine + 0.02);
}
```

### Progress & State
- Maintain a `state` object: `{ chunks, chunkMeta, embeddings, retrievedChunks, composedQuery, answer, steps }`
- Each button enables the next one in the pipeline upon success
- `setStepProgress(n)` updates the top progress bar: steps 1..n-1 become green ✓, step n becomes blue active
- `setStatus(id, true)` marks the Process Status row as done (green)

### Toast Notifications
- Fixed bottom-right, auto-remove after 3s
- Types: success (green), error (red), info (blue)

---

## Interaction Details
- All buttons show a spinner while processing (simulated 600–900ms setTimeout delay)
- Clicking a chunk in Document Chunks panel highlights it with orange border/background
- Clicking `jump` in embeddings table or `view` in retrieval results calls `highlightChunk(idx)` which scrolls the chunk into view and applies the highlight
- Character count updates live as user types in the document textarea
- Top N meta text in AI Answer header updates dynamically when Generate Answer is clicked

---

## Responsive
- Below 900px: single-column layout, progress bar labels hidden (circles only), bottom grid stacks vertically

---

## Footer
`Demo for learning RAG · Data stays in your browser · No API calls made`
centered, gray-400, 12px

---

## Important Constraints
- **No external JS libraries** (no jQuery, no React, no lodash)
- **No backend, no API calls** — all logic is pure client-side JS
- **Single file** — all CSS in `<style>`, all JS in `<script>`, no imports
- Google Fonts loaded via `<link>` tag is the only external resource allowed
- Data never leaves the browser
